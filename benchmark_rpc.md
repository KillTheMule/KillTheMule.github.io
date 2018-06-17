# Overview

I've been working on my Neovim rcp plugin [nvimpam](https://github.com/KillTheMule/nvimpam) for some time now, and while I've wanted to
improve the code and add 1 or 2 features, I took a detour for some benchmarks, and I want to show the results.

I'm not going through the details of the plugin here, but what it does is provide folds for a certain filetype, the question I was after
is "After I've computed the folds, what's the fastest way to communicate them to neovim?". The plugin is written in [Rust](http://rust-lang.org),
and therefore the benchmarks are in Rust, too. The code isn't complicated though, so you should be able to follow along if you've
programmed yourself. Also, some Lua and Viml is sprinkled in as well.

On the other hand, the code mostly just does the obvious thing, and you might want to skip to the benchmarks at the end.

### Naive approach

The main function to benchmark is

```rust
   pub fn resend_all(&self, nvim: &mut Neovim) -> Result<(), Error> {
     nvim.command("normal! zE").context("'normal! zE' failed")?;
 
     // TODO: use nvim_call_atomic
     for range in self.folds.keys() {
       nvim
         .command(&format!("{},{}fo", range[0] + 1, range[1] + 1))
         .with_context(|e| {
           e.clone().context(format!(
             "'{},{}fo' failed!",
             range[0] + 1,
             range[1] + 1
           ))
         })?;
     }
 
     Ok(())
   } 
   ```
   
   Let's go though it real quick, don't worry too much about the details. First line in the function sends `zE` to neovim, to delete all
   existing folds. Then we iterate over the existing folddata, and the ranges are the keys in this datastructure. After that, we send
   commands of type `:1,5fo` to neovim to create the folds (the `format` macro assembles the strings we're sending). Never mind everything
   else, but note that we're sending a new command for each fold. That's bound to be inefficient, isn't it?
   
   ### call_atomic
   
   So let's send it all in one batch. `nvim_call_atomic` provides a way to do so, in Rust this looks like
   
   ```rust
pub fn resend_all_atomic(&self, nvim: &mut Neovim) -> Result<(), Error> {
    let mut calls: Vec<Value> = Vec::with_capacity(self.folds.len() + 1);

    calls.push( vec![
        Value::from("nvim_command".to_owned()),
        vec![Value::from("normal! zE".to_owned())].into(),
      ].into(),
    );

    for range in self.folds.keys() {
      calls.push(
        vec![
          Value::from("nvim_command".to_owned()),
          vec![Value::from(format!("{},{}fo", range[0] + 1, range[1] + 1))]
            .into(),
        ].into(),
      );
    }

    nvim.call_atomic(calls).context("call_atomic failed")?;

    Ok(())
  }
  ```
  
  This is a bit more involed, since we need to use all those `Value::from` and `into()` calls to transform our strings into
  msgpack `Value`s manually (which imho is a bit of a shortcoming of [neovim-lib](https://github.com/daa84/neovim-lib) which I'm using).
  The essence is that we're assembling a `Vec` (if you don't know Rust, take it as an Array) callled `calls`, that itself contains 2-`Vec`s, which
  follow the pattern `(function, args)`. Our function is `nvim_command`, and in the line below this you're seeing the familiar command
  we've been using above, too. After assembling it we send `calls` to neovim via `nvim.call_atomic`.

   ### Using lua
   
   As you can see above, we're sending a bit of redundant payload along, since we're only ever using Neovim's function `nvim_command`,
   and we're sending it along for each fold again. Wouldn't it be better to only send the fold data to minimize our communication
   volume? One way to do it is to define the following function in `nvimpam.lua`, which is around for the plugin anyways:
   
   ```lua
  local function fold_em(folds)
      for _, fold in ipairs(folds) do
        command(fold[1]..","..fold[2].."fo")
      end
  end
```
   
   Cute, isn't it? It takes an array of 2-arrays of numbers, and then just runs the `command` calls to create those folds. Just as we
   did above from Rust, just with a bit less communication volume, and a bit more indirection through the lua interpreter.
   
   The Rust function to call this is
   
   ```rust
 pub fn resend_all_lua(&self, nvim: &mut Neovim) -> Result<(), Error> {
    let mut folds: Vec<Value> = Vec::with_capacity(self.folds.len() + 1);

    for range in self.folds.keys() {
      let v = vec![Value::from(range[0]+1), Value::from(range[1]+1)];
      folds.push(v.into());
    }

    let args = vec![Value::from(folds)];
    nvim.execute_lua("require('nvimpam').fold_em(...)", args)?;

    Ok(())
  }
  ```
  
  Note the use of `require` to load/use  the nvimpam lua module.
  
  But now, we're only shelving out to viml anyways, because that's what `nvim_command` does. How bout using viml directly?
  
  ### viml
  
  Let's just real quick source a viml file containing
  
  ```viml
  function! Fold_em(folds)
  for fold in a:folds
    execute fold[0].','.fold[1].'fo'
  endfor
endfunction
```

which does exactly the same thing as the lua code. From Rust we can call it like


```rust
pub fn resend_all_viml(&self, nvim: &mut Neovim) -> Result<(), Error> {
    let mut folds: Vec<Value> = Vec::with_capacity(self.folds.len() + 1);

    for range in self.folds.keys() {
      let v = vec![Value::from(range[0]+1), Value::from(range[1]+1)];
      folds.push(v.into());
    }

    let args = vec![Value::from(folds)];
    nvim.call_function("Fold_em", args)?;

    Ok(())
  }
  ```
  
  which just uses `nvim_call_function` instead of `nvim_execute_lua`.
  
  
  ### The benchmark
  
  This is very much Rust specific, so don't fret over it, I'm mainly providing it for reference. We're running it 4 times for each
  of the functions we defined above, the difference is really just this one function call you can see somewhat towards the end:
  
  ```rust  
#[bench]
fn bench_folds(b: &mut Bencher) {
  let (sender, receiver) = mpsc::channel();
  let mut session = Session::new_child_cmd(
    Command::new("neovim/build/bin/nvim")
      .args(&["-u", "NONE", "--embed"])
      .env_clear()
      .env("VIMRUNTIME", "neovim/runtime"),
  ).unwrap();

  session.start_event_loop_handler(NeovimHandler(sender));
  let mut nvim = Neovim::new(session);

  nvim.command("e files/example3.pc").expect("2");

  let curbuf = nvim.get_current_buf().expect("3");

  b.iter(|| {
    let mut foldlist = FoldList::new();
    let mut lines;
    curbuf.attach(&mut nvim, true, vec![]).expect("4");
    loop {
      match receiver.recv() {
        Ok(LinesEvent { linedata, .. }) => {
          lines = Lines::new(linedata);
          foldlist.recreate_all(&lines).expect("5");
          foldlist.resend_all(&mut nvim).expect("6");
          curbuf.detach(&mut nvim).expect("7");
          nvim.command("call rpcnotify(1, 'quit')").unwrap();
        }
        _ => break,
      }
    }
  });

  let _ = nvim.command(":qa!");
}
```
 
 ###  The result
 
  benchmak | result |variance
 ---------|--------|----
|test bench_folds        |26,301,579 ns/iter |+/- 4,581,534
|test bench_folds_atomic |  15,502,505 ns/iter |+/- 4,035,317
|test bench_folds_lua    |  15,187,042 ns/iter |+/- 3,967,311
|test bench_folds_viml | 15,198,539 ns/iter |+/- 3,855,720
 
 
 We are seeing a nice speedup from not doing so many call, but we are not really seeing a diffence by from tuning down the volume
 of the one communication call we still need. But here's a catch: The example file we're running this on just requires about 5 folds, 
 so there isn't much communication overhead (which really makes it all the _more_ surprising we're seeing any difference it all
 from reducing the number of calls).
 
 ### Alternative result
 This is from a file that requires a lot more folds, so we are enlargening the differences:
 
 benchmak | result |variance
 ---------|--------|----
 test bench_folds       | 293,494,416 ns/iter |+/- 107,315,251
 test bench_folds_atomic |   21,941,457 ns/iter |+/- 2,571,597
 test bench_folds_lua    | 22,725,711 ns/iter |+/- 2,751,634
 test bench_folds_viml |  17,184,134 ns/iter |+/- 3,678,667
 
 Whoaaa now that is a speedup! More than 10 times faster just by combining all the calls into one! I'll need to find a more 
 complicated file still!
 
 Nevertheless, what we're also seeing is that the viml version pulls slightly ahead. The difference to the lua version could be
 explained by the fact that the lua version calls out to viml anyways, so that contains an additional indirection. But  why is
 `call_atomic` a bit slower than viml? No idea on my side, but then again, I don't really know how all of that is implemented.
 
 Anyways, that's it for out little benchmark excurion. Comments, criticism? Let me know through the
 [neovim reddit](https://www.reddit.com/r/neovim/). Thanks for reading!
 
