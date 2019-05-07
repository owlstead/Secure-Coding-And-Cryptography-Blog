# On Wrapper Libraries and Classes

One of the great urges of any developer  is to simplify both their life and that of other developers. So what is better than helping other developers by hiding all the intricacies of, for instance, cryptography?

Well, actually a whole lot, so lets dive into why generic wrapper libraries are such a dangerous notion. Then list some reasons when they *are* applicable. And finally, what to replace them with, if anything.

As I'm a cryptographer, I'll focus on a wrapper library for a cipher, but many if not all of the concepts are applicable to wrappers around *any existing API*.

## The story of my "special" wrapper class

When I started developing on my first cryptography related job I got into a special projects team, especially setup to introduce cryptography into the products. Here I got into a team that basically consisted of developers specialized in security and C++ programming. These developers all used wrapper libraries or even their own (largely botched) crypto implementations.

However, I was starting as a Java developer, so I decided to create my own wrapper library. We were using smart cards so basically the DES class hid all the intricacies of using `Cipher.getInstance("DESEDE/CBC/PKCS5Padding")`. It basically operated only on a single message of bytes or text, shuffled all the exceptions under the table, removed `SecretKey` from view and so forth. There was a similar `AES` class as well, for when that was required.

Of course, such a class is very handy to use so I literally injected *anywhere* where triple DES was required. And this consisted of a lot of projects at that time. All was good. Well, mostly good. There were of course a few instances where an error should be caught instead of being turned into a `RuntimeException`. And hey, sometimes you *don't* want the IV to be prefixed, so you have to remove it again. Well, at least I didn't default to ECB, making all my ciphertext insecure.

However, after a while it dawned to me that, hey, the cipher `Cipher.update` methods were there *for a purpose*. And they operate on a range *within* an array for a reason. CBC mode was, in hindsight, not the best mode available to ciphers - what about using an authenticated cipher mode like GCM? Neither was triple DES of course. And, although triple DES is still considered *somewhat* secure for 3-keys / 192 bits I should really be using AES at some locations. I'm not a huge fan of cipher mobility, but at some places it would be a good idea to be able to switch between AES or triple DES. And then I was not even talking about all those API mistakes - what about stringly typed code that uses the platform encoding? What was I *thinking*? Since the ciphertext was now out there in either Windows-1252 encoding as well as UTF-8 there was no way to use *just* UTF-8 during decryption. Of course, there was no version identifier included with the ciphertext - mainly because there was no protocol description in the first place.

So I finally decided that enough was enough. The class needed to be removed from the equation all together. That's when I found out how productive I had been. The class was used just about everywhere; not just in the proof-of-concept code where it started it's life. Actually, some of the PoC code had turned into an actual product - nothing new there. And there was not an easy option to refactor it either by inlining the code - it would just add a layer of obfuscation while keeping the issues of using it.

The fix? *Rewrite all the code that was still in use, and freeze the code that was not in use anymore!*. This took me a few weeks of intermittent development on it, spending an hour of two between my other projects. Procuring the man-hours for such a project with management was an obvious no-go. After all, there was no new product to show afterwards. And I had to do it, because the other devs didn't know my style or what needed to change.

## My insights on wrapper libraries

So that's the story, now let's turn a bit more precise and list the insights about wrapper libraries:

 1. **Wrapper libraries and classes are used to wrap the *knowledge* of the developer *rather than the API itself*.** Generally, this happens at the time that the developer is still *learning about the subject*. I'll let you mull over of the implications of this statement privately; take your time.

 2. Although the idea is to simplify the code, generally it happens to *over*-simplify the code, which will come back to bite you if you ever need to refactor. For instance, my `DES` class didn't stream, so you'd need `Cipher` again to do anything of that kind.

 3. By letting the other devs use a proprietary wrapper library, you are hiding the generic API from them. They are learning the API  of your proprietary wrapper library instead. If they ever need to change your library, they need to learn *both* the wrapped library as well as your design and code style.

 4. Instead of a whole Internet full of experts, you've just reduced your number of experts to one: the designer of the wrapper class. Any change to the library will have to go through this one expert, unless you want to get into a shouting match after you missed this obscure feature that he introduced and what 10% of the projects rely on.

 5. Any mistakes in the output will be extremely hard to fix, especially if no protocol or protocol version has been defined. Are you *sure* that the wrapper always behaves as you expect it to? Because the plaintext is gone if a single bit of the key or original ciphertext is invalid.

 6. Standard API's are generally well defined, well tested and well documented. Mistakes will be made, but often those get fixed over time. Wrapper libraries on the other hand are often the project of a single developer. This will have a very negative effect on the quality of documentation *and* testing. In other words, maintenance is commonly a nightmare.

 7. Wrapper libraries are commonly *very* inflexible. Any change will reflect in *all* the projects where they are used. Changing just one function - for instance because of a bug - may require you to find *all* the usages of the library, and test *all* software that uses it. You did set up automated Unit testing, *right*?

Did you expect such a long list? If not, take some time and think about the thousands of wrapper classes on the Internet that just wrap `Cipher`.

## When do wrapper libraries make sense?

There are a few  requirements that should be fulfilled when writing wrapper libraries for production use:

 1. The technology of the generic API is too different to be used in a generic / high level way. If you have to convert to / from `char*` to `string::` before and after each call, then your code is going to be maintainability nightmare as well. This is the *use case*, the *why* of the library.

 2. You're expecting to do a whole lot with it. Creating a generic wrapper library is a bad idea if you only use it once, after all. If you use it once you just include the code into the project of the use case instead.

 3. The experience to create such a library is present. If this is the first time for a developer to use an existing API, then *this is not the time to develop that wrapper library*. Remember, you need to design, document and test this library very well.

 4. You're willing to setup a maintenance structure around the library. There should be people responsible for maintaining code after all, *especially* if it is security related. You want to be able to get a response if a (security) bug is found, after all.

In that sense my fellow C/C++ programmers possibly had a point in creating a wrapper for cryptography, as the underlying PKCS#11 library was hard to use directly from well formed C++. I'll however spare you the countless hours that a few of the senior developers spend on it (even *after* turning architect). Using PKCS#11 or the OpenSSL EVP API directly would probably have been less painful.

*Language specific wrapper libraries do usually make sense*. It is highly useful to be able to use the high level OpenSSL (EVP) functionality *within a higher level language than C/C++*. However, remember that many of the wrapper libraries *still* display a lot of the drawbacks listed in the insights. Fully wrapping, documenting and testing the OpenSSL functionality, including streaming, all the ciphers and such takes *a lot* of development effort even without counting the maintenance.

## What to replace the wrapper library with?

Nothing. Just use the API's that are available to you, and swallow some pride when handling that same exception again. If you must, create some *code practices* for the use of the library.

Crypto API's will require some learning curve, sure, but after that all the developers will know how to use the hopefully well defined API. Issues with it will become easier to work around, and your developers will have been trained to use it. With a bit of luck you can find new or external developers that also know about it.

Rather than developing the wrapper, spend your time defining *high level* API's that are required to implement *the specific use cases of a product*. Nobody is really interested in a `MySignature#verify`. However, `MyDocumentType#verify` makes a lot more sense, especially if the protocol and algorithms to use for  `MyDocumentType` are well described.

## What to do with that existing wrapper library?

Whatever you do, *don't let that wrapper get into your production code*. If it is already in there, *get it out now* or make sure that the code is frozen. Review it, set up a maintenance structure and don't let it proliferate into new projects if the wrapper is everywhere. Setup automated testing in case you do find a bug, and create a list which libraries are "infected" with it, so you can perform damage control when it is needed.

You could let the original developer do a presentation on the code in the wrapper library. This wrapper library probably started off as a learning project anyway. And the original developer may learn a lot him/herself by having to explain the underlying concepts. More than you would probably expect. There is nothing wrong with creating code for learning practice, after all. And please point to this post, or you'll have 10 wrappers libraries instead of just one.
