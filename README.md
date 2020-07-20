# sharkbite v0.04
This will help convert tunneled icmp traffic that is plaintext (mostly used for ctf challenges) back to tcp traffic for extraction and inspection.

THIS WILL MAKE NON ICMP TRAFFIC UNREADABLE.  ONLY USE ON COPIES OF PCAPS.

To use, you must have tshark and text2pcap installed.

Has been tested with custom plaintext .pcap and .pcapng files that had ICMP traffic thru a tunnel.

If there is extra charators in the beginning of data area of the icmp packet, you can add an offset as $2 when launching code.
  The offset needs to be number of bytes of padding x2 to get the correct offset.
  
A new folder will be created in the same folder as ran from that will contain the following:
    the folder has the same name as the .pcap it was ran againt with _cut added to the end.
      -the new pcap as created by code.
      -any files extracted by the code.
  
  If folder has no extracted files, then one or more of the following probly happened:
    -the icmp traffic had no files and it was being used by a different tunneling process.
    -the icmp traffic had padding in the data section of the icmp packet.  count and correct.
    -the icmp traffic had been security measures that encoded the frames and they are still encoded.
    - or other problems that I did not encounter.

