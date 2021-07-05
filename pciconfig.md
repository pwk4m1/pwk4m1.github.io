[home](/) [feedback](/feedback) [links](/links)

-----------------------------------------------------------------------------

## PCI 

	PCI spec feels pretty nice so far, quite clean. PCI Bus is a local
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

	So, the goal of mine  was to get device ID of each device. 
	If we take a look into the standardized part of configuration space, 
	we can see that Device ID & Vendor ID make up the first 32 bits. 
	This means, that we can grab device ID by reading the high 16 bits of 
	first config dword.

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
		out 	dx, eax
		add 	dx, 4     ; 0xCF8 + 4 = 0xCFC
		in 	eax, dx

	And here we go, it works :)

-----------------------------------------------------------------------------

## So then what?

	The next step would be to read and parse rest of the configuration 
	space, and then configure devices & space for them accordingly.
	However, that's a story for another night.

-----------------------------------------------------------------------------

