# Programming languages in W3S

...or wouldn't this all be easier if we rewrote it in Cobol...

## Authors

- [Hannah Howard]

## Some basics

- Currently, W3S is written almost entirely in Javascript, therefore, every dev on W3S must be able to write code in Javascript
- Javascript is not a language that is well suited for every function in W3S, particularly as we evolve to a more distributed network running on bare metal hardware, and potentially do blockchain programming
- Every additional language we add potentially adds team fragmentation. Therefore:
  - We should not add languages because we "like" them, or get excited by them, or cause they seem #SoHotRightNow
  - There must be a strong engineering driven reason to adopt a new language.
- Not every dev will be an expert in all our languages. But once we adopt a language, every dev must be willing to do work in that language if it's required. We support each other and teach each other to make work get done faster.
- Candidate languages worth considering adding are Go and Rust. In the future we will probably need to write some Solidity contracts.

## Thoughts on programming languages from Hannah

My baseline philosophy is no one should be super attached to the language they write code in. Moreover, I think everyone gets better as a developer the more experiences they get across programming languages. When I hire, I generally try to hire smart people who can reason through problems and trust them to learn what they need. I tend to think of the core craft of programming as being relatively static.

One thing I miss about in office work (one of the only things) is that working in person gives people more opportunities to teach each other through pairing and knowledge sharing. This is a super fast way to learn languages. If you find yourself on a ticket in a language you're less familiar with, I highly recommend pairing with a language expert. Seriously, ask for people's time to get fast rather than staying slow. And in the age of ChatGPT, even if you can't find a pair you can probably get functioning if need be. 

In my opinion, the actual reasons to choose a programming language almost always have nothing to do with the language syntax itself, or even the language's core philosophy about how to write code (if it has one). In fact, choosing a language for these reasons is probably almost an anti-pattern in the workplace.

Instead the best reasons to use a language come from unique capabilities unlocked by the compiler and toolchain, the runtime, the standard library, or the surrounding ecosystem. IMHO the selling point of Java is the JVM, not the Java language. The selling point of Erlang is how its scheduling model and message passing architecture enable reliable execution of highly concurrent distributed real time applications (i.e. chat applications like WhatsApp). 

As a project, we should add languages to W3S when we can do things with them that we either can't do or can't do very well in the languages we already write code with. That's the only reason to do so.

## Language analysis from a business and engineering perspective

### Javascript

- The single best reason to write Javascript is if you think that now or in the future you might run your code in a web browser. In our case, we know this is true for the w3up client.
- Modern javascript is a highly capable language, if you unify a your team on a single coding style, linter, tool chain, package manager, approach to typing, and transpilation system as needed. Unfortunately, almost every one of the above has multiple options and the ecosystem is an incoherent mess.
- Javascript's single threaded asynchronous programming model turns out to actually be pretty decent for writing web servers too.
- A lot of devs are used to deploying code on centralized services with built in javascript/node support and forget that distributing binaries is still the best way to get code to machines in a truly distributed system like the one we're building. Javascript's binary story kind of sucks. And, while JS runtime(s) are pretty optimized, JS is still pretty unpredictable in terms of memory usage, and to a lesser extent performance. Javascript, at least in its node.js/browser form, also has a pretty big size runtime(s).
- Every hiring manager loves javascript cause it's easy to hire javascript developers. The main problem with this approach is there are so many kinds of javascript development and ways to write javascript that hiring a javascript developer is not a gaurantee they'll be able to write javascript the way your team uses it.

#### Some Notes On Our Javascript

- Thankfully this team has unified on a coding style, linter, tool chain, package manager, and approach to typing. +1 on that.
- We have not however documented why we made the choices we did. We should do so ASAP. In particular types in JSDoc style comments and why we do it needs documentation. This includes our sporadic dropping into Typescript and why we do it. (For example: 'why is there an api.js and an api.ts and the js file just says `export {}` -- the answer is 'cause modules' but is EXTREMELY not clear)
- The w3up repo is huge, and honestly, even if you know javascript, there's a lot to get up to speed on. We should have a guide to the code.

### Go

- Go is a highly performant garbage collected language like Java, but is much simpler than Java
- Go has an extremely powerful concurrency model with green threads
- These two components together make Go one of the most powerful languages for writing servers of any kind. The challenge of server programming is not super fast compute, but super high concurrency execution. Go excels at this, which matters cause our project has a lot of servers involved.
- Go's ability to do binary distributions by default is a major plus for kind of project.
- By and large, people get stuff done quickly in Go because Go is extremely simple. It's very fast to learn, easy to read, and it all looks the same. The only problem is its extremely boring. (Note from Hannah: When I first started writing Go I hated it. I thought someone took the joy out of programming. Now I love it cause I get my joy from what I'm building, not my programming language)
- Lots and lots of Filecoin tooling and software is written in Go. In fact, W3S already has Go in it, we just don't maintain it -- it's called Spade
- Garbage collection and green threads present challenges for using Go over an FFI, or for compiling go to WebAssembly, though TinyGo has improved this story by reducing the runtime size.

### Rust

Caveat from Hannah: I am not a highly experienced Rust dev

- Rust is a better C/C++. Like definitely much better. This is huge cause no one wrote a better C/C++ for low level programming for 20 years.
- However, by and large, you should probably not write programs in Rust unless you'd consider writing them in C/C++. There is minimal evidence one can be as productive in Rust as higher level languages.
- In particular, you should probably not write web servers in Rust or any other asynchronous programs unless you have extra time on your hands which we don't.
- BTW, if you haven't programmed in C/C++ recently, you've probably forgotten that long compile times are a thing on large projects in low level languages. 
- Rust is essential for highly performant synchronous compute tasks. That's why people use it for smart contracts, hashing, and proofs.
- Rust has the best WebAssembly story of any language.

## Table Stakes For Considering A Language for W3S

In addition to having a compelling engineering or business reason, we have some other things to consider before we adopt a language:

- UCAN libraries and tooling especially for anything with a web interface
- IPLD/IPFS/Libp2p libraries (we may not always need libp2p going forward)

## Proposing a language to adopt

- The best time to propose a language to adopt is at the start of a new project
- Present a proposal to write a project in a new language via an RFC
   - Try to objectively present business and engineering tradeoffs of writing the project in a language we already support vs writing it in a new language
   - Identify gaps in language support or tooling related to our software stack and present a plan to address it
   - Ideally, make a case for why this will be useful for other projects going forward
   - Present a plan for bringing other team members up to speed on the language you propose to use. Usually the language proposer will have to captain the process of language education.
- Since it's a big decision, we should try to get input from all devs before making a decision at least till our team gets bigger
- Must have sign off from senior engineering leadership (for now Hannah and Alan)
