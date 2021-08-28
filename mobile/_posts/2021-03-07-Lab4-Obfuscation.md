# Lab 4 - JS Obfuscation
## The basics
Obfuscation is used to confuse the reversing process. This can be done by making unessesery complex loops that at the end of the day don't get used or return a value of "1" so that the "attacker" (in this case the person who is trying to reverse the app) wastes a lot of time, or to scare them off.

Obfuscation makes the code unreadable for humans but clearly readable to machines without using special functions or decodings. 

The example below shows a few ways of counting numbers and generating characters to print code.

	Obfuscation examples
	------------------
	# Numbers
	+![] 							// 0 - the number 0
	+!![] 				            // the number 1
	+!![] + +!![] 		            // the number 2
	
	# Letters
	([][[]]    +[]) 				// "undefined"
	\u0061                          // a
	\u006C                          // l
	\u0065                          // e
	\u0072                          // r
	\u0074                          // t
	
## Numbers
Counting numbers can be done by returning the value of a boolean. An empty array returns a `False` boolean value which correspondents to `0`. Putting a `!` in front of it makes it `True` which in turn is counted as `1`. So knowing this we an start counting numbers.

## Letters
An easy way to obfuscate letters you would type like "alert" is by using the unicode value of the letters instead of typing the letters. Browsers and JVM understand this as one and the same but it makes it a lot harder to read for humans. 
An other way of generating characters is by generating an error, turning that error into a string and extracting the characters from that string.

## Tips
- Look at the spacing, this is important to know `+!![]` -> this plus will transform the `True` to a `1`. If we want to count with this we can't just append another `+` to this. We need to seperate the values with a space. (Test it out see if it works).
- You can use as many `!` as you want infront of a empty list. However at the end of the day you'll still only have 2 possible values: `True` & `False
- If you need one space, you can add multiple. Just don't add a space when you can't add spaces.
	- spaces in (most) arrays are okay

## The lab
### Lab 1
Knowing this generate the JS code: `alert("fun")`. The only characters you are allowed to use are the ones mentioned in the "example".
- Sollution: `\u0061\u006C\u0065\u0072\u0074(([][[]]+[])[+!![] + +!![] + +!![] + +!![]] + ([][[]]+[])[+[]] + ([][[]]+[])[+!![]])`

### Lab 2
Knowing the code below, make an obfuscation that spells the word: `android`

	_ = ~[];

	_ = {
		a: {
			a: !![]+[], // "true"
			b: ![]+[],  // "false"
			c: []+{},   // "[object Object]"
			d: [][_]+[] // "undefined"
		},
		b: ++_,         // 0
		c: ++_,         // 1
		d: -~_++,       // 2
		e: -~_,         // 3
		f: -~++_        // 4
	};
	
- Allowed characters: `a, b, c, d, e, f`
- Allowed symbols: `_ . + [ ]`

Sollution:

	console.log(_.a.b[_.c] 	
		+ _.a.d[_.c]
		+ _.a.d[_.d] 
		+ _.a.a[_.c]	        
		+ _.a.c[_.c]
		+ _.a.d[_.f + _.c]
		+ _.a.d[_.d]);	