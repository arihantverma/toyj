## Takeaways form Chapter 1:
- > Because computers are dumb, pedantic beasts, programming is fundamentally tedious and frustrating.
- > Fortunately, if you can get over that fact—and maybe even enjoy the rigor of thinking in terms that dumb machines can deal with—programming can be rewarding
- > On top of that, it makes for a wonderful game of puzzle solving and abstract thinking.
- > Like human languages, computer languages allow words and phrases to be combined in new ways, making it possible to express ever new concepts.
- > Programming, it turns out, is hard. You’re building your own maze, in a way, and you can easily get lost in it.
- > When you are struggling to follow the book, do not jump to any conclusions about your own capabilities.* You are fine—you just need to keep at it*
- > Learning is hard work, but everything you learn is yours and will make further learning easier.
- > When action grows unprofitable, gather information; when information grows unprofitable, sleep.
  > — Ursula K. Le Guin, The Left Hand of Darkness
- Really liked the program's definition:
	- > it is data in the computer’s memory, and, at the same time, it controls the actions performed on this memory
- > The best programs are those that manage to do something interesting while still being easy to understand.
- > A sense of what a good program looks like is developed with practice, not learned from a list of rules.
- > To program early computers, it was necessary to set large arrays of switches in the right position or punch holes in strips of cardboard and feed them to the computer. You can imagine how tedious and error prone this procedure was.
- > It has made modern web applications possible—that is, applications with which you can interact directly without doing a page reload for every action
- ## Chapter 2
- > As soon as you no longer use a value, it will dissipate, leaving behind its bits to be recycled as building material for the next generation of values.
- `typeof` is actually an operator and not just a 'keyword'
- ```
  console.log(null == undefined);
  // → true
  console.log(null == 0);
  // → false
  ```
  
  That behavior is often useful. When you want to test whether a value has a real value instead of `null` or `undefined`, you can compare it to `null` with the `==` or `!=` operator.
- > When comparing strings, JavaScript goes over the characters from left to right, comparing the Unicode codes one by one.
-