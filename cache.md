[home](/) [feedback](/feedback) [links](/links)

-----------------------------------------------------------------------------

### Cache as Ram

	I got annoyed at using ROM as my stack (it's slow, requires bit of
	an effort to get working if doable at all), only using registers
	for ram init is also a bit of an effort.. So I took a look at
	coreboot & how they do stuff, instantly encountering this apparently
	old, but totally new for me concept of using Cache as ram.

	In theory, this is as simple as it sounds, configure MTRR subsystem
	and clear cache-disable & not-write through bits from cr0, then
	clear cache region to use & set stack pointer to CAR base address.

	In practise, this proved to be almost as simple, but I was bit too
	eager to implement this, skipped parts of intel's manual regarding
	Caching, and messed things up for quite a while..

-----------------------------------------------------------------------------

### Issues

	The setup part was quite straightforward, there aren't _too_ many
	resources, but old linuxboot docs, coreboot sourcecode + docs, 
	mtrr specs & overall intel system programming guide combined were
	plenty enough to get this done. You might notice heavy *cough*
	inspiration *cough*, aka copying of code parts from coreboot &
	linuxboot.

	However, exit from CAR mode wasn't easy at all.. Well in reality
	it was, but I made a silly mistake, which only affected how my 
	disk reads worked.  
	
	Before I implemented CAR, disk read worked reliably, even though
	admittedly slowly in ATA 28b pio mode. As soon as I added 
	Cache As Ram operations into my code, disk reads started suddenly
	returning only 00 or FF bytes. Disk identification etc. still
	worked correctly, as did all PCI operations, communication with 
	interrupt controller, etc...

	This was super confusing, for the longest time I believed I had a bug
	in my disk driver that is only triggered somehow with this changed
	entrycode.

	Then, I was reminded that oftentimes disk reads can make use of Cache,
	which led me to doublecheck my CAR exit code... I had disabled 
	MTRR subsystem, cr3 was 0, but cr0 still had
	cache-disable bit set as 0... CPU was using non-configured cache.

	After changing cache-disable to 1 in cr0, everything worked just
	fine again.

-----------------------------------------------------------------------------

### Implementation

	I wont be going too deeply in detail here, linuxboot & coreboot
	docs explain this stuff way better than I do, however here's a short
	walkthrough of how to enable cache-as-ram on intel i386&x86 machines.

	As always, don't copy-paste my code & use it as is, it may break stuff.

-----------------------------------------------------------------------------


	%define CACHE_AS_RAM_BASE       0x08000000
	%define CACHE_AS_RAM_SIZE       0x2000
	%define MEMORY_TYPE_WRITEBACK   0x06
	%define MTRR_PAIR_VALID         0x800

	%define MTRR_PHYB_LO            (CACHE_AS_RAM_BASE | \
						MEMORY_TYPE_WRITEBACK)
	%define MTRR_PHYB_REG0          0x200
	%define MTRR_PHYB_HI            0x00

	; mask
	%define MTRR_PHYM_LO            ((~((CACHE_AS_RAM_SIZE) - 1)) | \
						MTRR_PAIR_VALID)

	%define MTRR_PHYM_HI            0xF
	%define MTRR_PHYM_REG0          0x201

	%define MTRR_DEFTYPE_REG0       0x2FF
	%define MTRR_ENABLE             0x800

 	; setup MTRR base
        mov     eax, MTRR_PHYB_LO       ; mtrr phybase low
        mov     ecx, MTRR_PHYB_REG0     ; ia32 mtrr phybase reg0
        xor     edx, edx
        wrmsr

        ; setup MTRR mask
        mov     eax, MTRR_PHYM_LO
        mov     ecx, MTRR_PHYM_REG0
        mov     edx, MTRR_PHYM_HI
        wrmsr

        ; enable MTRR subsystem
        mov     ecx, MTRR_DEFTYPE_REG0
        rdmsr
        or      eax, MTRR_ENABLE
        wrmsr

        ; enter normal cache mode
        mov     eax, cr0
        and     eax, 0x9FFFFFFF
        invd
        mov     cr0, eax

        ; establish tags for cache-as-ram region in cache
        mov     esi, CACHE_AS_RAM_BASE
        mov     ecx, (CACHE_AS_RAM_SIZE / 2)
        rep     lodsw

        ; clear cache memory region
        mov     edi, CACHE_AS_RAM_BASE
        mov     ecx, (CACHE_AS_RAM_SIZE / 2)
        rep     stosw

        xor     ax, ax
        mov     ss, ax
        mov     esp, (CACHE_AS_RAM_BASE + CACHE_AS_RAM_SIZE)

	; Done, now cache is used as stack,.. hopefully.
	; atleast it works on my machine!

