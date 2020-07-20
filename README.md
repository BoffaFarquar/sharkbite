# sharkbite v0.04
This will help convert tunneled icmp traffic that is plaintext (mostly used for ctf challenges) back to tcp traffic for extraction and inspection.  Uses tshark and text2pcap directly.

THIS WILL MAKE NON ICMP TRAFFIC UNREADABLE.  ONLY USE ON COPIES OF PCAPS.

To use, you must have tshark and text2pcap installed.

Has been tested with custom .pcap and .pcapng files that had ICMP traffic thru a tunnel.

If there is extra charators in the data area of the icmp packet, you can add an offset as $2 when launching code.
  The offset needs to be number of bytes of padding x2 to get the correct offset.
  
A new folder will be created in the same folder as ran from that will contain the following:
    the folder has the same name as the .pcap it was ran againt with _cut added to the end.
      the new pcap as created by code.
      any files extracted by the code.
  If folder has no extracted files, then one or more of the following probly happened:
    the icmp traffic had no files and it was being used by a different tunneling process.
    the icmp traffic had padding in the data section of the icmp packet.  count and correct.
    the icmp traffic had been security measures that encoded the frames and they are still encoded.
    
###################################################################################################################    
here is a direct copy and paste of the code.

#!/bin/bash
### 
### sharkbite_v0.04.sh
###
### syntax:		./sharkbite_v0.04.sh <pcap file> <bytes of data.data to remove x 2>
###	example:	./sharkbite_v0.04.sh Ping.pcap 10 --- 5 bytes x 2 = 10 char
###
### this will convert plaintext tcp traffic over a icmp tunnel back to tcp traffic.
### it only works on icmp.  It will destroy all other traffic in pcap.
### the second argument is only used if a offset in the data.data requires
### addional bytes to be removed.
### 
### v0.03 changes to make v0.04
###		dumped required fields all at once into one file before part 1
###		moved crop of extra data to before part 1 when using $2 argument
###		changed formulas to be computed faster
###		changed new bash calls to occur differently in current bash (faster)
###		
####################################################################################

### Define Variables and start time
dataoffset=$2		### use number of bytes x 2 - used in Part 1 - Read each packet into shark
((dataoffset++))	### need to add 1 to value do to new way of cutting awk $3 if needed
start_time="$(date -u +%s)"
nudgeup=`tput cuu 1`

### checking for the pcap argument
if [ -z "$1" ]; then echo "NO arguments supplied"; exit 1; fi
if [ -f "$1" ]; then echo "Processing file $1"; echo; else echo "File $1 does not exist."; exit 1; fi

### remove and create folder with name of pcap with _cut as a suffix
if [ -d "$1_cut" ]; then rm -fr $1_cut; fi
mkdir $1_cut

### extract pcap parts to file and load packets total
tshark -r $1 -T fields -e eth.addr -e eth.type -e data.data | awk -v a="$dataoffset" '{ $3 = substr($3, a); print }' > $1_cut/zfile1_all.txt
packetstotal=$(tshark -r $1 | tail -n 1 | awk '{print $1}')

############### Start part 1 - reading of packets, modifying, and writing to zfile3*
input="$1_cut/zfile1_all.txt"
flc=1
while IFS= read -r line
do
	### Name the packet being worked on.
	echo -en "\033[1A"
	echo "Processing packet  $flc  of  $packetstotal  ...   "

	### remove char(s) that do not translate to packets and format
		echo $line | awk  '{ $2 = substr($2, 7); print }' | sed 's/://g' | sed 's/,//g' | sed 's/ //g' | sed 's/.\{32\}/& /g'|sed 's/ /\n/g' | sed 's/.\{2\}/& /g'  > $1_cut/zfile1_$flc.txt
	
	((flc++))
done < "$input"
################ End part 1

################ Start part 2 - reading of zfile1* adding hex locations as prefix on every line and writing to zfile3*
for (( x=1; x<=$packetstotal; x++ ))
do 
	### Notify of packet written to hexdump zfile2*
	echo -en "\033[1A"
	echo "Writing packet     $x  of    $packetstotal  to hexdump file....."
		
	flc=1
	input="$1_cut/zfile1_$x.txt"
	while IFS= read -r line
	do
		hexlocation=$(($flc*16-16))
		finallocation=$(printf "%06x" $hexlocation)
		
		echo "$finallocation $line" >> $1_cut/zfile3all.txt
		((flc++))
	done < "$input"
done	
############### End part 2

############### Start part 4 - use zfile3 to create pcap
### Takes all of the zfile3* file and writes it to a pcap
text2pcap $1_cut/zfile3all.txt $1_cut/$1_cut.pcap -q
############### End part 4

############### Start part 5 - extract files from pcap
### uses tshark to extract files that are found in the pcap
tshark -nr $1_cut/$1_cut.pcap --export-objects http,$1_cut > /dev/null
############### End part 5

### Clean Up
rm -fr $1_cut/zfile*

### Time finished and diff
end_time="$(date -u +%s)"
elapsed="$(($end_time-$start_time))"
echo -en "\033[1A\033[K"
echo "Total of $elapsed second(s) elapsed for single process."
