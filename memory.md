[home](/) [feedback](/feedback) [links](/links)

-----------------------------------------------------------------------------

### Malloc()

	Implementing malloc properly is a bit of an effort I hear...
	That's why I chose not to :)

	However, I needed a way to allocate memory as I need it, and free
	it once I'm done with it,, and as usual, instead of taking some 
	already working code, I chose to write it all from scratch.

-----------------------------------------------------------------------------

### Memory blocks

	I chose to divide all of my memory into "blocks". in practise this
	means that I start with one big contiguous chunk of memory, and then
	start splitting it & re-combining it back together with malloc&free.

	Coming up with a way to prevent memory from being totally exhausted
	and fragmented immediately was slightly difficult, eventually I came
	up with following design:

	Each of memory 'blocks' must have a header that has following info:

		- Signature text, 'free' or 'used'
		- Size of the block
	
	Malloc() function goes through the memory space reserved for it 
	looking for block that's marked as free. Once it finds the 'free'
	signature, it checks if the block is big enough, allocates needed
	area of the block, and creates new, smaller one that contains 
	leftover memory.

	This kind of approach would not be very usable in say, Operating 
	System context, but for single-threaded BIOS it's good enough...

-----------------------------------------------------------------------------

### Free

	Free'ing the allocated memory is seemingly simple, you just replace
	the 'used' signature with 'free', right? NO! that would lead to
	situation where we have gazillion few-bytes large free memory blocks
	and malloc could never succeed as none of these would be big enough..

	What I chose to do, was to loop through whole memory region of malloc
	and preform following operation:

      +=========> Have we reached end of memory? == YES ==> Exit
      | 	         |
      | 		 NO
      | 		 |
      | 		 V
   Next block <== NO === Is the memory marked as free? 
      ^			 |
      |			 Yes
      |			 |
      |			 V
      +========== NO === Is the next block marked as free?
      |	                 |
      |			 Yes
      |			 |
      |			 V
      |			 Combine these two blocks
      +==================V

	This kind of function prevents memory fragmentation. As with malloc,
	this is very slow approach, but it's good enough for this kind of 
	application.

-----------------------------------------------------------------------------

### Putting it all together

	Apart from malloc() and free() there is one more function related,
	which simply builds the first block, marks it free, and sets size of
	the block to be our whole malloc'able memory region.

