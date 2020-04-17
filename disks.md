[home](/) [feedback](/feedback) [links](/links)

-----------------------------------------------------------------------------


## Delightful dive into world of ATA

	Writing driver for ATA disks was fun in it's own way, there was open 
	documentation & free access to standards, as opposed to the horrific
	mess that eg. VGA is... And as this driver will be operating as part
	of very minimalistic BIOS, it can be made to be pretty dumb & use
	PIO mode. 

	Even if PIO is very cpu exhausting & slow way to read disks, it's 
	required to be supported by all ATA-compliant drives. This makes my
	life a lot easier, as I don't have to think of all differend kinds of 
	ATA disks out there.

	The PIO mode basicly works over IO bus of cpu, instead of being memory
	mapped IO. This restricts the speed of disk operations to 16MB/s at
	highest, and prevents other processes running during the disk opers.
	That isn't really an issue here, because our BIOS has no actual reason
	to be multi-threaded, and we're only interested in reading the first
	512 bytes of our boot disk for now. 


### Flowcharts and stuff

	Even if PIO mode is pretty straightforward and simple, I wasn't very
	familiar with disk driver developement. I kept on trying to write
	the driver w/o proper planning for atleast a  week or so, before giving
	up & listening to people who told me I'd need to plan the driver with
	pen and paper to make it more clear..  Below is a small 'flowchart' 
	I eventually came up with, after throughly reading through osdev wiki
	about ata pio for 10th time.

![Image](/img/ata_flowchart.jpeg | width=100)

	So now I had a clear idea about how-to implement the driver, and goal
	for each of the functions I'd be writing. There were few more weird
	things to realize & got right, such as the ~400ns delays to make sure
	that there's enoguh time for these slow disks to push correct voltages
	onto the bus. Overall, there were quite a few weird quirks during the
	project that I didn't take into account & spent time banging my head
	to wall with:

		- There needs to be that +400ns delay I mentioned when running
		  outside emulators/vms

		- Cache might need to be manually flushed, some drives don't
		  flush the cache for you, which might lead to write operations
		  invisibly failing. 

		- Believe it or not, disks might have bad sectors & writes may
		  fail, all kinda weird stuff happens :)

### Disk IO

	As I mentioned earlier, this driver was coming for BIOS, so it doesn't
	need to be very fancy nor does it need to support multi-threading etc.
	Ofc, it'd be nice to make it 'proper' at somepoint and get rid of
	polling disks for data. Currently disk read/write works according
	the ascii flowchart below:

```

 			+---------------------- YES ---+
			V			       |
		Disk select works?  -- NO --> Retry after reset works?
			|			       |
			YES 			       NO
			| 			       |
			| 			       V
			V			Disk has hung. exit
 ABORT <---- NO ------	Disk R/W params are sane? 
				|
				YES
				|
				V
 +-----------+--------> Read/Write operation ok? -- YES --> DONE
 ^	     ^ 			|
 |	     |	 		NO
 |           |			|
 |    TMP Bad block <-- YES -- Write works?
 |	do reset                |
 | 				NO
 |				|
 |				V
 +---- Do reset <-- NO -- Other blocks work?
 ^				|
 |				YES
 |				|
 |				V
 +----------------- Permanently bad block, chane the block
```

	I think that the flowchart started with only checks of wheter 
	disk select works or not, and parameter check, but when when testing
	stuff, I eventually added more and more stuff to code+flow chart..

	The next steps for the driver would probably be to use PCI for disk
	detection incase non-standard IO location is used, and change the
	read operations to be interrupt-driven instead of current way of 
	polling.. We'll see when I have motivation to finish the driver /
	abstraction for pci to make that happen.. Hopefully during next weeks.

### Floating disks
	
	I'm not entirelly sure how cpu side of ata bus is wired to achieve
	this, and I don't think it'd be too hard to find out, but anyway for
	now it remains a mystery.. ata bus with no disks 'floats' at constant
	+5 volts (or whatever voltage your machine uses to indicate high), 
	resulting to reads from bus to return all 1's. Specs tell us it's
	enough to check if bits 1 and 2 of status register are both 1s, as this
	should never be the case, but I think it's more neat to just do

```
	mov 	dx, ATA_STATUS_REGISTER
	in 	al, dx
	cmp 	al, 0xFF
```

	Because if the bus floats & as result all we read is 1s, we can just
	compare against 1111 1111 (FF) just as well.. This way we know that
	there really is nothing connected, but with status bits 1 and 2 being
	1's, but others still being eg 0, we can assume *something* is either
	connected to the bus, or broken.

