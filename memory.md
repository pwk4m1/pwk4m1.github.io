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

	As is, malloc() from the BIOS could be as follows in pseudo C:

		void *
		malloc(short size)
		{
			/* declarations */
			ret = 0;
			size = size + sizeof(MEM_PTR_STRUCT);
			while (ptr_current < MEM_END) {
				if (ptr_current->signature != MEM_FREE) {
					ptr_current += ptr_current->size;
					continue;
				}
				if (ptr_current->size < size) {
					ptr_current += ptr_current->size;
					continue;
				}
				/* Don't leave small blocks here. */
				if (ptr_current->size == (size + 0x10)) {
					ptr_current->signature = MEM_USED;
					ret = ptr_current;
					break;
				} else {
					ptr_next = ptr_current + size;
					ptr_next->signature = MEM_FREE;
					ptr_next->size = (
							ptr_current->size - \
							size);
					ptr_current->signature = MEM_USED;
					ret = ptr_current;
				}
			}
			return ret;
		}

	My implementation also has some checks, such as panic if 'out of sync',
	aka if we encounter block that has invalid or no signature. I chose not
	to include it into the pseudo C example, mostly to make it even clear
	that the pseudo code there is not something you should copy-paste
	directly to any project.. it wont work as is.

	The out of sync I mentioned happened multiple times when fixing the 
	implementation, as first I didn't take into account the fact that 
	memory 'block' headers also need space for them, so everything was 
	working seemingly fine until the moment I used _whole_ block,
	overwriting the next header, and then got panic at next malloc().

-----------------------------------------------------------------------------

### Free

	Free'ing the allocated memory is seemingly simple, you just replace
	the 'used' signature with 'free', right? NO! that would lead to
	situation where we have gazillion few-bytes large free memory blocks
	and malloc could never succeed as none of these would be big enough..

	What I chose to do, was to loop through whole memory region of malloc
	and preform following operation:

		1.) Get next memory block
	
		2.) Are we at the end of memory? if yes, exit

		3.) Is the memory block free? if not, goto 1

		4.) Is the memory block after this free? if not, goto 1

		5.) Combine this and next memory block, then goto 1

	This kind of function prevents memory fragmentation. As with malloc,
	this is very slow approach, but it's good enough for this kind of 
	application.

	In practise, this could be implemented in pseudo code roughly
	like this:

		while (ptr_current < MEM_END) {
			if (ptr_current->signature != MEM_FREE) {
				ptr_current += ptr_current->size;
				continue;
			}
			ptr_next = ptr_current + ptr_current->size;
			if (ptr_next->signature != MEM_FREE) {
				ptr_current = ptr_next;
				ptr_current += ptr_current->size;
				continue;
			}
			/* Two blocks are both free, combine them */
			ptr_current->size += ptr_next->size;
			ptr_current->size += sizeof(memory_block_header);
		}

-----------------------------------------------------------------------------

### Putting it all together

	Apart from malloc() and free() there is one more function related,
	which simply builds the first block, marks it free, and sets size of
	the block to be our whole malloc'able memory region.

	And now, finally I can do:

		mov 	cx, some_struct_size
		call 	malloc
		test 	di, di
		jz 	.malloc_errored
		push 	di

		; ...

		pop 	di
		call 	free
	

-----------------------------------------------------------------------------
