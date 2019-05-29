---
layout: post
title:  "Multithread TCP Server in C - A beginner's journey"
subtitle: "The frustration and learning progress of a beginner dealing with a multithread TCP server"
date:   2019-05-29 11:07:00 -0700
background: '/img/posts/tcp-bg.jpg'
---

## Context

During our intership, we (me and other interns) were given a task related to network. The task is as follow:

>Using C/C++ write a server and client with the following functionalities:
> - Server:
>   - Generate an array with length from 100 to 1000, all the elements in the array are random, we are free to choose the range of element, for my implementation i choose it to be from 1 to 100.
>   - Accept connections from a maximum of 10 clients.
>   - Each time a client send a request, give them the next number from the array.
>   - Once all elements are given to the clients, tell them that the server has run out of ball when the next request comes in.
>   - The server now listen for the record of all elements the client received from it previously.
>   - The server then calculate the sum of elements from each client, rank them and send the whole ranking to each client.
> - Client:
>   - Send request to the server and receive an element from the array, sort all the element it currently has and log them down to a file.
>   - When received `out of ball` signal, send the whole file record of elements back to the server.
>   - Wait and receive the ranking from the server.


Prior to this internship, I have not taken any courses on network programming nor multi thread programming so this task is quite a challenge. Before doing this task, we have been given some materials on the subject but reading them alone withouth context or concrete example is not really helpful so this this task is the first time I have my hand on all this task.

Complete code for both server and client can be found at [this git hub repository](https://github.com/ptnghi/simpe-tcp-multithread-server/blob/master/startclient.sh)

## Intial search and problems

So as any sane and clueless person would do, I googled "TCP multithread server" and found some codes. Here's the part I found [here](https://dzone.com/articles/parallel-tcpip-socket-server-with-multi-threading)

```c

#include<stdio.h>
#include<stdlib.h>
#include<sys/socket.h>
#include<netinet/in.h>
#include<string.h>
#include <arpa/inet.h>
#include <fcntl.h> // for open
#include <unistd.h> // for close
#include<pthread.h>
char client_message[2000];
char buffer[1024];
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
void * socketThread(void *arg)
{
  int newSocket = *((int *)arg);
  recv(newSocket , client_message , 2000 , 0);
  // Send message to the client socket 
  pthread_mutex_lock(&lock);
  char *message = malloc(sizeof(client_message)+20);
  strcpy(message,"Hello Client : ");
  strcat(message,client_message);
  strcat(message,"\n");
  strcpy(buffer,message);
  free(message);
  pthread_mutex_unlock(&lock);
  sleep(1);
  send(newSocket,buffer,13,0);
  printf("Exit socketThread \n");
  close(newSocket);
  pthread_exit(NULL);
}
int main(){
  int serverSocket, newSocket;
  struct sockaddr_in serverAddr;
  struct sockaddr_storage serverStorage;
  socklen_t addr_size;
  //Create the socket. 
  serverSocket = socket(PF_INET, SOCK_STREAM, 0);
  // Configure settings of the server address struct
  // Address family = Internet 
  serverAddr.sin_family = AF_INET;
  //Set port number, using htons function to use proper byte order 
  serverAddr.sin_port = htons(7799);
  //Set IP address to localhost 
  serverAddr.sin_addr.s_addr = inet_addr("127.0.0.1");
  //Set all bits of the padding field to 0 
  memset(serverAddr.sin_zero, '\0', sizeof (serverAddr.sin_zero));
  //Bind the address struct to the socket 
  bind(serverSocket, (struct sockaddr *) &serverAddr, sizeof(serverAddr));
  //Listen on the socket, with 40 max connection requests queued 
  if(listen(serverSocket,50)==0)
    printf("Listening\n");
  else
    printf("Error\n");
    pthread_t tid[60];
    int i = 0;
    while(1)
    {
        //Accept call creates a new socket for the incoming connection
        addr_size = sizeof serverStorage;
        newSocket = accept(serverSocket, (struct sockaddr *) &serverStorage, &addr_size);
        //for each client request creates a thread and assign the client request to it to process
       //so the main thread can entertain next request
        if( pthread_create(&tid[i], NULL, socketThread, &newSocket) != 0 )
           printf("Failed to create thread\n");
        if( i >= 50)

        {
          i = 0;
          while(i < 50)
          {
            pthread_join(tid[i++],NULL);
          }
          i = 0;
        }
    }
  return 0;
}
```

I took that as reference, fixed some minor syntax error, and quickly whiped up a test server with it. It was a simple echo server at first where it just send back whatever I send it. I used netcat for client since it is quite simple and everything worked fine. 

Pushing ahead, I implement part of the logic of the given task to this framework and boom we have a ball-spitting machine. 

Quite confident with what I accomplished, I wrote the client's code and a start script to start things in parallel. (The start script can be found [here](https://github.com/ptnghi/simpe-tcp-multithread-server/blob/master/startclient.sh)) This is where the server code start to show some **real problems.**

The first and really minor problem is that the integer `i` in the loop never got increased as new threads are created, thus overwritng the previous one. When the number of client actually reaches 50 and we need to wait for the previous ones to finish, it will only wait for one thread but not the rest. 

The second one and most serious problem is sending the reference to `newSocket` to the new thread. On paper, the new value would be passed to the new thread and convert to a socket file descriptor for each new client. However experiment showed otherwise. (It is a shamed that I don't have the time to revert my code to that state and show you the actual result.) Different threads have the same socket descriptor passed to them, reuslting in one or a few client not receiving any data and hangs forever.

This is caused by a context switch happen right before the thread copy the arguments to a local variable. Imagine a sequence like this

>New connection comes in -> New thread created but have not copy the value of file descriptor -> Context switch back to the main thread -> new connection comes in -> The value in newSocket whoes reference got passed to threads got changed -> 2 different threads now receive the same socket descriptor.

So I decided to change this part a little bit. Instead of passing the `newSocket` as reference in, I change the variable itself to a pointer, before every accept, I malloc for it a new memory block. This way, the thread handling clients will never receive the same socket descriptor and overlap each other. Implementation detail can be found in the repository mentioned above.

## Synchronization 

I just toook a course on Operating System prior to this intership where the content focuses heavily on solving synchronization problem. Therefore all of the synchronization stuff is not much of a problem for me during this task. However there are a few tips I can give:

- Protect shared resource, always, race condition is a mess.
- Remeber to unlock after lock a mutex, especially when you break out of a loop.
- Try to minimize the critical section


## Problem with TCP packages

I just toook a course on Operating System prior to this intership where the content focuses heavily on solving synchronization problem. Therefore all of the synchronization stuff is not much of a problem for me during this task. However there are a few tips I can give:

- Protect shared resource, always, race condition is a mess.
- Remeber to unlock after lock a mutex, especially when you break out of a loop.
- Try to minimize the critical section

TCP is quite a a reliable protocol with all the acknowledgement procedure and stuff however it still have some gotcha that a complete beginner like me have no idea about.

The first one is maximum package size. TCP cannot send a package that is too large. When we try to send the result file (which is about 2-3KB) the end of the file is dropped. To fix this, we have to split the content into a few 1024 bytes package. To accomodate this logic change, both the server and client's code have to incorporate partial send and read.

The second problem is a very sneaky one. If you send two packages that are really small in size in in very short time span, they will get truncated into one packge and send as a whole. This caused some serious rage in our room as things get sent all over the place while no one know the reason until one of our mentor actual came and explained it. In the end, no concrete solution were found but a work around is implemented instead. Some of my friends put a very tiny sleep before to `send()` call so that they will not get sent as one. As for me I make sure I have enough stuff done between two `send()` calls.


## Blocking and Non-blocking

We read about block and non-blocking in the materials but never realized its importance until actually get smashed by it. So mostly everything in Linux is blocking and `accept()` is no exception.

After fixing all of the previous problem. The server and client worked fine but there's still one problem: My server won't exit after finish everything but just stay there. At the time, I didn't know that `accept()` would block so I fire up gdb and investigate where the program is haninging. After seing that it actually hangs at `accept()`, I google some more and realize that `accept()` will block until a connection comes. More googling lead me to [this great tutorial](https://jameshfisher.com/2017/04/05/set_socket_nonblocking/) on how to set `accept()` to be non block. 

Once that is done, everything worked the way I wanted and it was time for me to rejoice this victory and write this blog. Thank you for reading and happy coding.


