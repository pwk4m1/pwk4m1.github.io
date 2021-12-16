[home](/) [feedback](/feedback) [links](/links)

-----------------------------------------------------------------------------

## Callbacks from long 64bit mode to real 16bit mode.

	Whilst working on building a bootloader for OS project, encountered
	the fact that it's quite common nowadays to have images that
	have loaders for both UEFI and legacy bios loaders. As far as what
	short web searching told me, the loaders are still separate. It works
	but we thought about making sort of 'unified' loader, aka having 
	platform-specific 'abstraction api'/1st stage loader, and unified 
	second stage loader that works exactly the same for both UEFI and 
	legacy bios devices.

		The legacy bios provides api by having set interrupt handlers
	corresponding to various function, so that the bootloader can do
	swint to call the bios api. The apis mostly assume they're called
	in 16bit real mode, and they often do return to 16bit real mode 
	regardless of how the entry was done. The UEFI provides 64bit long
	mode. This means there's quite a difference between how legacy bios
	and UEFI apis work. This means we need to provide way for the 2nd 
	stage loader to return from 64bit long mode to 16bit real mode.
	
		Idea for how to implement this was to provide simple callback
	function for 2nd stage that stores command registers, msrs, etc. 
	related to runmode. Then once the bios api call is done we can simply
	restore everything to how it was & return to 2nd stage from the 
	real mode stuff.

-----------------------------------------------------------------------------

### The actual preface/abstract

	There's essentially, as far as I know, two ways to implement the
	jumps back and forth between runmodes:

		1: Use iret
		2: Do what iret does manually.

	The iret might be easier and faster and objectively better way to
	implement the switches but I didn't get it to work reliably. The 
	issue I encountered with iret was that I messed something up somehow
	which lead to the cpu being in 32bit, not 16bit mode. The iret issue
	should be quite trivial to debug but I chose to fallback to something
	I know how to do.

		The way I chose to go was 2nd one, by manually backing up
	all the registers, and moving:

		1: from 64bit long mode to 32bit protected mode
		2: from 32bit protected mode to 16bit protected mode
		3: from 16bit protected mode to 16bit real mode

	The 3rd part of the moving back to realmode is something I overlooked
	for quite a while. However, after debugging for minute or two with
	Qemu+gdb this was easy to note and all.

-----------------------------------------------------------------------------

### Switch from 64bit long mode back to real 16bit mode

	To return back from long mode to protected 32bit mode we only have to
	disable LM bit from EFER MSR, disabling PG bit of command register 0, 
	jumping to CS32:offset. Then disable PAE from command register 4 
	and flush tlb by setting command register 3 to 0. 
	Finally, set segments to DS32 and you've successfully returned to 
	32bit protected mode.

		The next step is to return to _protected_ 16bit mode.
	this bit is quite a lot more simple than previous one, all we have to
	do is to jump to CS16:offest, and set segments to 16 bit PM ones.

		To finally return to real mode, you'll have to disable PM bit
	from control register 0, jump to 0:offset, setting segments as you 
	need, and be done with it.

-----------------------------------------------------------------------------

### Returning from 16bit to 64bit mode

	Now that you're in 16 bit real mode, you can call your bios api as
	you wish. Once that's done, the return is just business as usual to
	return back to 64bit mode. I performed this and return with having
	backed up command registers, msrs, etc. and by simply returning the
	backed up values to corresponding regs. No issues whatsoever.

-----------------------------------------------------------------------------

### Put it all together / code

	I wont be showing my code for this quite yet, as it still needs to
	be tidied up. However, following is a small PoC for returning to
	real mode from long mode.


	bits	32
	callback_16_from_64:
		cli

		; start by disabling long mode stuff
		mov 	ecx, 0xC0000080
		rdmsr
		and 	eax, 0xfeff
		wrmsr

		mov 	eax, cr0
		and 	eax, 0x7fffffff
		mov 	cr0, eax

		; jump back to 32 bit mode
		jmp 	gdt.code32:.pm32

	.pm32:
		; disable PAE
		mov 	eax, cr4
		and 	al, 0xdf
		mov 	cr4, eax

		; flush tlb
		mov 	eax, 0
		mov 	cr3, eax

		; Set segments to 32 bit ones
		mov 	ax, gdt.data32
		mov 	ds, ax
		mov 	es, ax
		mov 	gs, ax
		mov 	fs, ax
		mov 	ss, ax

		; disable protected mode
		jmp 	gdt.cs16:.pm16

	bits 	16

	.pm16:
		mov 	ax, gdt.ds16
		mov 	ds, ax
		mov 	es, ax
		mov 	ds, ax
		mov 	gs, ax
		mov 	fs, ax
		mov 	ss, ax

		mov 	eax, cr0
		and 	al, 0xfe
		mov 	cr0, eax
		jmp 	0x0000:.rm16

	.rm16:
		xor 	ax, ax
		mov 	ds, ax
		mov 	es, ax
		mov 	gs, ax
		mov 	fs, ax
		mov 	ss, ax

		mov 	ax, 0x0003
		int 	0x10

		cli
		hlt
		jmp 	$ - 2

	rm_idt:
		dw 	0x03ff
		dd 	0x0000

