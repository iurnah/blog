---
title: "Block and Non Block IO Socket"
date: 2015-04-08 
categories:
  - C
  - Python 
tags:
  - socket
draft: false
toc: true;
---

I have received an email from Honeynet project, they give me a code test for google summer of code 2015. The problem is

>to write one program in python and another one in c to read arbitrary input from one socket and write to another for each.

This problem is much harder than it appears. I am writing a post for it here.

# Solutions in C

1. using the system `select()` function.
2. set the sockets to be non-blocking and use polling.
3. software interrupt whenever input on a socket arrives.
4. spawn off a separate thread to handle I/O. `sproc()` in Saloris
5. use `XtAppAddInput()` in X/Xt code.

## The input socket

We call the small program proxy. The function of the proxy is to read bytes from a TCP or UDP socket sockIn, and write arbitrary bytes to another socket sockOut. For TCP, at the input end, in order to read bytes from sockIn, sockIn should have bytes available to write to proxy, and connection to proxy. There is a design decision need to be made: should the proxy actively connect to the sockIn, or the proxy listen for sockIn to connect?

__Implementation 1: Proxy Passively listen and read:__

| 		SockIn Client 		| 	ProxyIn Server	|
|---------------------------|-------------------|
| `connect()`, `gets()`, `write(ProxyIn)` | `listen()`, `accept()`, `read(ProxyIn)`, `write(ProxyOut)`|

__Implementation 2: Proxy actively connect and read:__

| 		SockIn Server 		| 	ProxyIn Client	|
|---------------------------|-------------------|
| `accept()`, `gets()`, `write(ProxyIn)` | `connect()`, `read(ProxyIn)`, `write(ProxyOut)`|

From the above tabulated function call for different implementation, you can see the complexity is similar. All the used function calls are to establish the connection, read arbitrary bytes from sockIn.

I will start with the first implmentation for the input end. The proxy should keep running for ever, altering between the following states:

>state 1. listening for `sockIn`,

>state 2. accepted sockIn,

>state 3. read x bytes from `sockIn`,

>state 4. wait for `sockOut` write ready.

>state 5. write x bytes to `sockOut`.

>state 6. complete write to `sockOut`, return to state 1.

## The output socket
`sockIn` connect to `sockOut` and write data to it. So the whole picture would be looks like this:
```
                      sockredirect.c
                     +----------------+
                     |bind            |
                     |listen   connect|
stdin---->sockIn---->|accept     write|---->sockOut---->stdout
                     |read            |
                     |                |
                     +----------------+
```

The implementation of this scheme in c:

```c++
/**************************************************************************
 * This program read arbitrary bytes from sockIn and write it to sockOut 
 *
 * USAGE: 
 *   compile: gcc sockredirect.c -o sockredirect
 *   ./sockredirect <proxyIn_IP> <proxyIn_port> <sockOut_IP> <sockOut_port>
 *   create input socket: run "nc <proxyIn_IP> <proxyIn_port>" in second terminal
 *   create output socekt: run "nc -l <sockOut_port>" in third terminal
 * 
 * test: 
 *   type characters in second terminal
 * exit: 
 *   when in the terminal run sockredirect, press Ctrl-C 
 *
 ***************************************************************************/ 
#include <sys/types.h>
#include <sys/socket.h>
#include <netdb.h>
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

#define MAXLISTENQ 10
#define BUFFERSIZE 32

void Die(char *mess) { perror(mess); exit(1);}

void redirectBytes(int sockIn_fd, char *ip, char *port){
	char buff[BUFFERSIZE];
	long readbytes = 0;
	long totalbytes = 0;
	int write_fd;
	int conn_write_fd; 

	struct sockaddr_in writeservaddr;//the socket this program write to, it is server for the output end.
	if((write_fd = socket(AF_INET, SOCK_STREAM, 0)) < 0){
		Die("problem creating ProxyOut!\n");
	}

	bzero(&writeservaddr, sizeof(writeservaddr));
 	writeservaddr.sin_family = AF_INET;
	writeservaddr.sin_addr.s_addr = inet_addr(ip);
 	writeservaddr.sin_port = htons(atoi(port));

	if((readbytes = recv(sockIn_fd, buff, BUFFERSIZE, 0)) < 0){
		Die("Problem receiving initial bytes from sockIn!\n");	
	}

	printf("connecting to SockOut...\n");
	if(connect(write_fd, (struct sockaddr *)&writeservaddr, sizeof(writeservaddr)) < 0){
		printf("ProxyOut fail to connect SockOut!\n");
		close(write_fd);
		return;
	}else
		printf("Output socket connected. SockOut(%s:%s)\n", ip, port);

	while(readbytes > 0){

		if(send(write_fd, buff, readbytes, 0) != readbytes){
			Die("Failed to send bytes to SockOut!\n");
		}else{
			totalbytes += readbytes;
		} 
		printf(" %5ld bytes of data fowarded\n", totalbytes);

		if((readbytes = recv(sockIn_fd, buff, BUFFERSIZE, 0)) < 0){
			Die("Problem receiving initial bytes from sockIn!\n");	
		}
	}
}

int main(int argc, char **argv){
	int read_fd; 
	int conn_read_fd; 
	struct sockaddr_in readcliaddr;	//the socket this program read, it is client
	struct sockaddr_in readservaddr;//this program's input end, it is a server when read bytes
	uint32_t readsockport;
	char readsockip[INET6_ADDRSTRLEN];
	socklen_t Rcliaddrlen;

	if(argc != 5){
		printf("USAGE: \n  proxy <proxyIn_IP> <proxyIn_port> <sockOut_IP> <sockOut_port>\n");
		printf("  create input socket: run \"nc <proxyIn_IP> <proxyIn_port>\" in second terminal\n");
		printf("  create output socekt: run \"nc -l <sockOut_port>\" in third terminal\n\n");
		printf("test: \n  type characters in second terminal\n");
		printf("exit: \n  when in the proxy terminal, press Ctrl-C \n");
		return 0;
	}
	// create a socket that listen and accept the readcli to read bytes 
	if((read_fd = socket(AF_INET, SOCK_STREAM, 0)) < 0){
		Die("problem creating ProxyIn!\n");
	}
	// configure address, and bind
	bzero(&readservaddr, sizeof(readservaddr));
	readservaddr.sin_family = AF_INET;
	readservaddr.sin_addr.s_addr = inet_addr(argv[1]);
	//readservaddr.sin_addr.s_addr = inet_addr("10.33.13.144");
	readservaddr.sin_port = htons(atoi(argv[2]));

	if(bind(read_fd, (struct sockaddr *) &readservaddr, sizeof(readservaddr)) <0 ){
		Die("Problem binding ProxyIn\n");
	}

	if(listen(read_fd, 10) < 0){
		Die("Problem listen on ProxyIn!\n");
	}

	while(1){
		printf("waiting for connecting input socket...\n");
		conn_read_fd = accept(read_fd, (struct sockaddr *)&readcliaddr, &Rcliaddrlen);
		if(conn_read_fd > 0){
			inet_ntop(AF_INET, &readcliaddr.sin_addr, readsockip, sizeof(readsockip)), 
			readsockport = ntohs(readcliaddr.sin_port);
			printf("input socket connected. sockIn(%s:%d)\n", readsockip, readsockport);
		}else{
			printf("Problem accept to sockIn !!!\n");
		}

		redirectBytes(conn_read_fd, argv[3], argv[4]);
	}
	close(read_fd);
}
```

Note we could also just use one server socket. This implementation have to only use one socket for the redirection. In this scenario, both sockIn and sockOut works as a client to connect the program. sockIn and sockOut can operate in full-duplex mode.

# Solutions in Python
Here We give the corresponding python implementation:

```python
'''
USAGE: 
 proxy <proxyIn_IP> <proxyIn_port> <sockOut_IP> <sockOut_port>
 create input socket: run "nc <proxyIn_IP> <proxyIn_port>" in second terminal
 create output socekt: run "nc -l <sockOut_port>" in third terminal
'''
from socket import *
import sys

def redirectBytes(sock, ip, port):
	sockout = socket(AF_INET, SOCK_STREAM)	
	print "Connecting to SockOut...\n"
	data = sock.recv(32)
	sockout.connect((ip, int(port)))
	print 'Output socket connected. sockOut: (\'{}\', {})'.format(ip, port)
	total = 0 
	while data:
		sockout.send(data)
		total += len(data)
		print total, " bytes of data forwarded\n" 
		data = sock.recv(32)

	sock.close()
	sockout.close()

if len(sys.argv) != 5:
	print __doc__
else:
	sock = socket(AF_INET, SOCK_STREAM)
	sock.bind((sys.argv[1], int(sys.argv[2])))
	sock.listen(5)
	while 1:
		print "waiting for connecting input socket...\n"
		newsock, client_addr = sock.accept()
		print "input socket connected. sockIn:", client_addr
		print "\n"

		redirectBytes(newsock, sys.argv[3], sys.argv[4]);

```

# Testing the program

## 1. Bash `exec`
__commands:__

1. create socket(client): `exec file-descriptor<>/dev/tcp/IP-or-hostname-here/port`
3. `echo "hello socket" >&3 #write to the socket`
4. `cat <&3		#read from the socket`
2. `exec 3<&-	#close for read`
2. `exec 3>&-	#close for write`

## 2. testing scheme

1. run the proxy from first terminal,
2. creat a _sockIn_ from second terminal,
2. creat a _sockOut_ from third terminal,
3. write arbitrary bytes to the sockIn in the second terminal
4. read arbitrary bytes from the sockOut from third terminal


# References:

1. http://man7.org/linux/man-pages/man7/socket.7.html
2. http://www.gnu.org/software/libc/manual/html_node/Sockets.html#Sockets
3. http://www.lowtek.com/sockets/select.html
4. http://www.faqs.org/faqs/unix-faq/programmer/faq/