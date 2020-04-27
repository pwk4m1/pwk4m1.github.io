[home](/) [feedback](/feedback) [links](/links)

-----------------------------------------------------------------------------

## PCI 

	PCI spec feels pretty nice so far, sane and clean. PCI Bus is a local
	computer bus that connects multiple different devices, buses & such
	to cpu in kind of unified, simple way. The devices connected to PCI bus
	appear to busmaster as if they'd be directly connected to it's own bus
	and are generally assigned into processors address space.

	All PCI deviecs are required to have configuration space, which in 
	practise is a bunch of registers that contain information regarding
	the device. The values found in config space may contain IDs, Base
	address registers (BARs), Expansion ROM Base addresses, interrupt lines
	and pins, etc.

	So, goal for my day was to get PCI enumeration to work & get ID for
	every device that's connected to the PCI bus of the machine.

-----------------------------------------------------------------------------

## Configuration space

	As I mentioned above, we have config space full of cool stuff that does
	help us a ton when writing software to interact with devices connected
	to PCI bus. Accessing the PCI configuration space happens, atleast on
	intel platforms, by accessing Configuration Space Address and 
	Configuration Space Data I/O ports 0xCF8 and 0xCFC. From here on I'll
	be referring address port as CSA, and data port as CSD.

	This is a legacy method of accessing PCI devices, but as I'm not quite
	sure wheter or not memory mapped configuration can be used at this 
	point, and as we know for sure that access over I/O ports works always
	I'll be sticking with it for now.

	This method of accessing configuration space is called CAM, coming
	from Configuration Access Mechanism, and it allows us to read 256 
	bytes of devices address space indirectly via CSA and CSD. 
	The interaction happens by first writing address we want to interact
	with to CSA, and then reading or writing CSD port. The addresses to 
	interact with are calculated in following way:

	Intel:
	   (bus << 16) | (device << 11) | (function << 8) | offset | 0x80000000

	AMD:
	   (offset & 0xF00) << 16) | (bus << 16) | (device << 11) | \
	       (function << 8) | (offset & 0xff) | 0x80000000

-----------------------------------------------------------------------------

## Enumerating

	So, the goal was to get device ID of each device. If we take a look
	into the standardized part of configuration space, we can see that
	Device ID & Vendor ID make up the first 32 bits. This means, that we
	can grab device ID by reading the high 16 bits of first config dword.

	The enumeration of all the devices in Pseudo C could look something
	like this:

	uint32_t ids; /* vendor & device id pair */
	uint16_t did; /* device id */
	for (bus = 0; bus < 256) {
		for (slot = 0; slot < 32; slot++) {
			ids = get_pci_config(bus, slot, 0, 0);
			did = (uint16_t)(ids & 0xffff0000);
		}
	}

	Seems simple enough, I think, now just have to implement that and
	pci config reads in assembler.. Which ended up being, well, nice and
	pleasant, but it got me over-confident and thus I made some silly 
	mistakes..

-----------------------------------------------------------------------------

## PCI config read

	As we now know, we first need to write address to CSA port, then
	read value from the address from port CSD. We also know how-to
	build addresses. I chose to use following memory structure to keep 
	track of bus/slot/offset/function we're at:

	byte offset | content
	          0 | bus
	          4 | slot
	          8 | function
	         12 | offset

	As you see, I am lazy & take 4 bytes for each of the values, even 
	though it could be made a lot smaller.. I'll rewrite that eventually.
	The address building in assembler was fairly easy when I finally 
	realized what I was doing wrong :) was accidentally doing right 
	shifts when i was supposed to shift left.

	The code goes kinda like this:

		mov 	ecx, dword [offset]
		mov 	ebx, dword [function]
		mov 	edx, dword [bus] ; we'll get to slot later
		mov 	eax, ecx 
		shl 	ebx, 8
		shl 	edx, 0x10
		or 	eax, 0x80000000
		or 	eax, ebx
		mov 	ebx, dword [slot] ; told you
		shl 	ebx, 0xB
		or 	eax, ebx
		or 	eax, edx

	So, we got first part of the reads done, rest is easy.. we now have
	our address in eax register, we can simply output it to CSA, and then
	read dword back in from CSD:

		mov 	dx, 0xCF8
		out 	dx, exa
		add 	dx, 4     ; 0xCF8 + 4 = 0xCFC
		in 	eax, dx

	And here we go, it works :)

-----------------------------------------------------------------------------

## So then what?

	The next step would be to read and parse rest of the configuration 
	space, and then configure devices & space for them accordingly.
	However, that's a story for another night.

-----------------------------------------------------------------------------

```
; BSD 3-Clause License
; 
; Copyright (c) 2020, k4m1 <k4m1@protonmail.com>
; All rights reserved.
; 
; Redistribution and use in source and binary forms, with or without
; modification, are permitted provided that the following conditions are met:
; 
; * Redistributions of source code must retain the above copyright notice, 
;   this list of conditions and the following disclaimer.
; 
; * Redistributions in binary form must reproduce the above copyright notice,
;   this list of conditions and the following disclaimer in the documentation
;   and/or other materials provided with the distribution.
; 
; * Neither the name of the copyright holder nor the names of its
;   contributors may be used to endorse or promote products derived from
;   this software without specific prior written permission.
; 
; THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
; AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
; IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR 
; PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR 
; CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, 
; EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO,
; PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR 
; PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF 
; LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING 
; NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
; OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
; 

%define __PCIDC_HDR_SIZE 0x9a

; ======================================================================== ;
; array used for storing pointers to pci device headers
; ======================================================================== ;
pci_dev_ptr_array:
	times 32 db 2

pci_dev_cnt:
	db 0

; ======================================================================== ;
; build address to read pci config word from. see pci_config_readw
; to see how-to setup memory at [si]
; ======================================================================== ;
__pci_config_build_addr:
	push 	ebx
	push 	ecx
	push 	edx

	mov 	ecx, dword [si+12]
	mov 	ebx, dword [si+8]
	mov 	edx, dword [si]
	mov 	eax, ecx
	shl 	ebx, 8
	shl 	edx, 0x10
	or 	eax, 0x80000000
	or 	eax, ebx
	mov 	ebx, dword [si+4]
	shl 	ebx, 0xb
	or 	eax, ebx
	or 	eax, edx	

	pop 	edx
	pop 	ecx
	pop 	ebx
	ret


; ======================================================================== ;
; Helper functions, mainly use these for developing additional logic based on
; PCI bus
; 
; ======================================================================== ;

; ======================================================================== ;
; read 1 byte from pci configuration space
; requires:
;	[si] 	= bus
; 	[si+4]  = slot
; 	[si+8]  = function
; 	[si+12] = offset
; returns:
;	al = config byte
; ======================================================================== ;
pci_config_readb:
	push 	bp
	mov 	bp, sp
	push 	dx
	push 	eax

	call 	__pci_config_build_addr
	mov 	dx, 0xCF8
	out 	dx, eax
	pop 	eax
	add 	dx, 4
	in 	al, dx

	pop 	dx
	mov 	sp, bp
	pop 	bp
	ret

; ======================================================================== ;
; write pci configuration byte
; requires:
;	[si]    = bus
; 	[si+4]  = slot
; 	[si+8]  = function
; 	[si+12] = offset 
;	al      = config byte to write
; ======================================================================== ;
pci_config_writeb:
	push 	bp
	mov 	bp, sp
	push 	dx
	push 	eax
	call 	__pci_config_build_addr
	mov 	dx, 0xCF8
	out 	dx, eax
	pop 	eax
	out 	dx, al
	pop 	dx
	mov 	sp, bp
	pop 	bp
	ret

; ======================================================================== ;
; read pci config word, requires:
;	[si] 	= bus
; 	[si+4] 	= slot
; 	[si+8] 	= function
; 	[si+12] = offset
; returns:
;	ax = config word
; trashes:
;	high 16 bits of eax
; ======================================================================== ;
pci_config_readw:
	push 	bp
	mov 	bp, sp

	call 	pci_config_readb
	mov 	ah, al
	call 	pci_config_readb

	mov 	bp, sp
	pop 	bp
	ret

; ======================================================================== ;
; read pci config dword, requires:
;	[si] 	= bus
; 	[si+4] 	= slot
; 	[si+8] 	= function
; 	[si+12] = offset
; returns:
;	eax = config dword
; ======================================================================== ;
pci_config_readd:
	push 	bp
	mov 	bp, sp

	call 	pci_config_readw
	shl 	eax, 16
	call 	pci_config_readw

	mov 	bp, sp
	pop 	bp
	ret

; ======================================================================== ;
; Write pci command, return status
; requires:
;	[si] 	= bus
; 	[si+4] 	= slot
; 	[si+8]  = anything, but allocated
; 	[si+12] = anything, but allocated
;	ax = config word to write
; returns:
;	ax = config word
; trashes:
;	high 16 bits of eax
; ======================================================================== ;
pci_write_cmd:
	push 	bp
	mov 	bp, sp
	push 	dx
	push 	ax

	mov 	dword [si+12], 0x04 ; offset to command word
	mov 	dword [si+8],  0x00
	call 	__pci_config_build_addr

	mov 	dx, 0xCF8
	out	dx, eax	 	; write out the address to use

	pop 	ax
	add 	dx, 4
	out 	dx, ax 		; write out the command word

	sub 	dx, 4 		; read the status
	mov 	dword [si+12], 0x06
	call 	__pci_config_build_addr
	out 	dx, eax
	add 	dx, 4
	in 	ax, dx

	mov 	sp, bp
	pop 	bp
	ret

; ======================================================================== ;
; add found pci device header to our list of pci devices
; sets carry flag on error.
; assumes SI to point to bus/slot/fun/off struct
; ======================================================================== ;
pci_add_device:
	clc
	push 	bp
	mov 	bp, sp
	pusha

	mov 	cx, __PCIDC_HDR_SIZE
	add 	cx, 2
	call 	malloc
	test 	di, di
	jz 	.error_no_memory

	mov 	al, byte [pci_dev_cnt]
	mov 	bx, 2
	mul 	bx
	add 	ax, pci_dev_ptr_array
	mov 	bx, ax
	mov 	word [bx], di

	movsd
	movsd
	xor 	cx, cx
	.loop:
		call 	pci_config_readw
		stosw
		add 	cx, 2
		cmp 	cx, (__PCIDC_HDR_SIZE - 8)
		jne 	.loop
		
	.done:
		popa
		mov 	bp, sp
		pop 	bp
		ret
	.error_no_memory:
		stc
		mov 	si, __pci_msg_no_memory
		call 	serial_print
		jmp 	.done

; ======================================================================== ;
; enumerate all pci devices & store info of them to ram, add pointer to
; header to pci_dev_ptr_array
; ======================================================================== ;
pci_init:
	push 	bp
	mov 	bp, sp
	pusha

	; allocate space for our bus,slot,function,offset structure
	mov 	cx, 16
	call 	malloc
	test 	di, di
	jz 	.error

	xor 	eax, eax
	mov 	si, di
	stosd
	stosd

	mov 	dword [si+12], 0x00
	.loop:
		call 	pci_config_readw
		cmp 	ax, 0xFFFF
		je 	.get_next_slot

		push 	ax
		push 	si
		mov 	si, __pci_msg_found_dev_id
		call 	serial_print
		call 	serial_printh
		pop 	si
		pop 	ax
		call 	pci_add_device
		jc 	.done
		mov 	al, byte [pci_dev_cnt]
		inc 	al
		mov 	byte [pci_dev_cnt], al
		cmp 	al, 6 ; we support up to 6 devices for now
		je 	.done

	.get_next_slot:
		mov 	eax, dword [si+4]
		inc 	eax
		mov 	dword [si+4], eax
		cmp 	eax, 32 ; maximum of 32 slots in pci bus
		jne 	.loop

	.get_next_bus:
		mov 	dword [si+4], 0
		mov 	eax, dword [si]
		inc 	eax
		mov 	dword [si], eax
		cmp 	eax, 256 ; theoretical max of 256 buses
		jne 	.loop
	.done:
		popa
		mov 	sp, bp
		pop 	bp
		ret
	.error:
		mov 	si, __pci_msg_no_memory
		call 	serial_print
		jmp 	.done

; ======================================================================== ;
__pci_msg_no_memory:
	db "PCI INIT FAILED, NOT ENOUGH MEMORY", 0x0A, 0x0D, 0
__pci_msg_found_dev_id:
	db "PCI DEVICE FOUND, VENDOR ID: ", 0

; ======================================================================== ;

```
