# Week 5 assignment — Put a limit on the tunnel

Student ID (last 4 digits only): **2071**

Write each section in your own words, about your own runs. Quote the evidence files in
`assignment/evidence/` that support each point. Delete the instructions in `<!-- -->` as you go.

## 1. Analysis: the loop


to create a tunnel loop we have to modify the handle 2 on s2 so its action will ouput on port1 instead of port 3 . We use this command 

`sh ./p4cli s2 "table_modify MyIngress.myTunnel_exact MyIngress.myTunnel_forward 2 1"` 

the number of packet increase a lot . in the ipv4 path in tunnel.p4 it does decrement the ttl but it doesn't stop the loop in this program . Because they are no code that check whether the ttl has become 0 and drop the packet . Instead after ttl become 0 , the value will underflow from 0 to 0xff and packet continue to loop over .


## 2. Design: the hop limit

Field size i add bit <8> hop_limit which will have a range from 0 - 255 . I use this number because this match the width of IPv4 ttl .

The default starting value in send.py in 64 but can be configurable using --hops argument . This is large enough for any realistic multi-hop campus or data-center topology while still terminating forwarding loop. 

its decrement in myTunnel_forward function as before design but it will check the value if the hop limit = 0 in my MyIngress apply function . If the hop limit = 0  it will drpo the packet . 

We have to change in Parser , Deparser , send.py and receive.py . 


## 3. Implementation

In tunnel.p4. 

1. extended header of myTunnel by adding hop_limit as third field . 
2. update action myTunnel_forward to decrement hop limit
3. update the apply function to drop packet if hop limit = 0 

send.py
1. Import ByteField
2. added --hops argparse and passed the hop limit into MyTunnel 

receive.py
1. Import ByteField
2. add ByteField , hop limit default value 
3. add --hops argparse 


## 4. Evidence: it works


So for the hop limit we put 10 in the argument so as there will be 2 hops to get from h1 to h3 . It decrease and the hop limit become 10 
```
Received (tunnelled, dst_id=2) from 10.71.0.1 to 10.71.0.3, TTL 64: 'hello h2', hop_limit=8
Listening on h3-eth0...
Received (tunnelled, dst_id=3) from 10.71.0.1 to 10.71.0.3, TTL 64: 'hello h3', hop_limit=8
```


Yes plain ipv4 still work 
```
mininet> h1 ping -c 2 h3
PING 10.71.0.3 (10.71.0.3) 56(84) bytes of data.
64 bytes from 10.71.0.3: icmp_seq=1 ttl=62 time=3.95 ms
64 bytes from 10.71.0.3: icmp_seq=2 ttl=62 time=3.04 ms
```

Yes the loop end . We can see the packet is not increasing and yes it match my design . 

```
mininet> sh grep -c "Processing packet received" logs/s1.txt logs/s2.txt
logs/s1.txt:936
logs/s2.txt:932
mininet> sh sleep 5; grep -c "Processing packet received" logs/s1.txt logs/s2.txt
logs/s1.txt:936
logs/s2.txt:932
```

## 5. Evaluation


a) i predict that the s2 will extract 4 bytes of myTunnel header while s1 will emit a 5-byte Tunnel header . so s2 would mis-parse the start of the inner ipv4 header and corrupt the packet . 

What actually happen
1. s1 sends the packet with all 5 bytes.
2. s2 reads only 4 bytes as the tunnel header. It doesn’t know a 5th byte exists.
3. Next, s2 expects the IP header. But the 5th byte (hop_limit) is still waiting, so s2 starts reading the IP header one byte too early. It treats the hop_limit byte as the first byte of the IP header. Everything after that is shifted by one.
3. s2 still forwards the packet to h2. That works because dst_id is in the first 4 bytes, where s2 expects it.
4. Before sending the packet out, s2 does its usual checksum update on the IP header. Since it’s reading the wrong bytes, it writes the checksum into the wrong place. This overwrites one real byte of the packet, the one that says “this is UDP”.


b) Safe Roll Out
Option 1 : New EtherType Version. 
Assign new EtherType (0x1213) to a 5-bytes myTunnel_v2 header while keeping 0x1212 for the 4 bytes myTunnel_v1 header

Rollout step : 
1. Upgrade al switches and receivers to support both 0x1212 and 0x1213 in their parser and ingress pipeline . 
2. Make the sender to encapsulate using 0x1213 
3. Remove 0x1212 support in a later maintenance cleanup


Option 2 : Make before break
and use control plane to drain the traffic onto the redundant path while upgrading switch one path at a time . 



The first option guarantee zero mis-parsed header and 0 packet drop during transition but we have to maintain 2 header type and parses state in switch hardware. The Second option , this require redundant multi-path topologies . For this topology , option 1 is the safest 


c) control plane analyses the forwarding table statically before the traffic is sent . It catches the forwarding loops proactively without wasting data-plane bandwidth . it can detect unreachable destination which the data plane cannot catch 

data plane it operate on live packet at runtime . it stop the loop that occur because of the controller update multiple switch asynchronously or out of band runtime edit . 

In real network , we nee dboth because control plane verify and prevent bad configuration from being deploy while data plane act as a fail-safe so bug or anything unexpected can be catch . 

d) 
1. extracting extra 8 bit use additional Packet Header Vector storage per packet across the pipeline stages
2. checking the hop limit = 0 will use small comparator and use ALU . but it doesn't require SRAM or TCAM memory 
3. Add 1 extra byte on the wire per tunnel packet make the line bandwith overhead increase . 

## AI use

Tools used : Gemini
How AI was used : 
1. The Gemini was used to understand given code  which is tunnel.p4 
2. Gemini help fix correct code syntax . this occur at send.py , receive.py and tunnel.p4 . Where it autosuggest on how to put correctly after i type the syntax wrongly
3. Gemini also use to explain part of the lab that is confusing  .  Gemini explain on how does the mixed happen and where to look in the logs . 



## Evidence files

| File | What it shows |
|---|---|
| `evidence/loop-before.txt` | The loop with your finished lab program: pass counts growing |
| `evidence/delivery.txt` | Tunnelled delivery with the hop limit, and a plain ping |
| `evidence/loop-after.txt` | The same loop with your hop limit: pass counts that stop |
| `evidence/mixed.txt` | The mixed rollout: s1 new, s2 old |
