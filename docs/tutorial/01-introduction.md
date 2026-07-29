const express = require("express");
const http = require("http");
const { Server } = require("socket.io");
const cors = require("cors");

const app = express();
app.use(cors());

const server = http.createServer(app);
const io = new Server(server, {
  cors: { origin: "*" },
});

// Stockage en mémoire des rooms
const rooms = {};

io.on("connection", (socket) => {
  console.log("Nouvelle connexion:", socket.id);

  // ---- CRÉER UNE ROOM ----
  socket.on("create-room", (callback) => {
    const roomCode = generateCode();
    rooms[roomCode] = {
      host: socket.id,
      videoId: "",
      isPlaying: false,
      timestamp: 0,
      queue: [],
      users: [socket.id],
    };
    socket.join(roomCode);
    console.log(`Room créée: ${roomCode}`);
    callback({ success: true, roomCode });
  });

  // ---- REJOINDRE UNE ROOM ----
  socket.on("join-room", (roomCode, username, callback) => {
    const room = rooms[roomCode];
    if (!room) {
      return callback({ success: false, error: "Room introuvable" });
    }
    socket.join(roomCode);
    room.users.push(socket.id);

    // Prévenir les autres qu'un nouveau arrive
    socket.to(roomCode).emit("user-joined", {
      socketId: socket.id,
      username,
    });

    // Envoyer l'état actuel au nouveau
    callback({
      success: true,
      state: {
        videoId: room.videoId,
        isPlaying: room.isPlaying,
        timestamp: room.timestamp,
        queue: room.queue,
      },
    });
  });

  // ---- SYNCHRONISATION VIDÉO ----
  socket.on("sync-video", (roomCode, data) => {
    const room = rooms[roomCode];
    if (!room || room.host !== socket.id) return;

    room.videoId = data.videoId;
    room.isPlaying = data.isPlaying;
    room.timestamp = data.timestamp;

    // Diffuser à tous les autres dans la room
    socket.to(roomCode).emit("video-update", {
      videoId: data.videoId,
      isPlaying: data.isPlaying,
      timestamp: data.timestamp,
    });
  });

  // ---- AJOUT À LA FILE D'ATTENTE ----
  socket.on("add-to-queue", (roomCode, video) => {
    const room = rooms[roomCode];
    if (!room) return;

    room.queue.push(video);
    io.in(roomCode).emit("queue-update", room.queue);
  });

  // ---- RETIRER DE LA FILE D'ATTENTE ----
  socket.on("remove-from-queue", (roomCode, videoId) => {
    const room = rooms[roomCode];
    if (!room) return;

    room.queue = room.queue.filter((v) => v.videoId !== videoId);
    io.in(roomCode).emit("queue-update", room.queue);
  });

  // ---- RÉACTION ----
  socket.on("send-reaction", (roomCode, reaction) => {
    socket.to(roomCode).emit("reaction", {
      socketId: socket.id,
      emoji: reaction.emoji,
      position: reaction.position,
    });
  });

  // ---- MESSAGE CHAT ----
  socket.on("send-message", (roomCode, message) => {
    socket.to(roomCode).emit("chat-message", {
      socketId: socket.id,
      text: message.text,
      username: message.username,
      timestamp: Date.now(),
    });
  });

  // ---- DÉCONNEXION ----
  socket.on("disconnect", () => {
    console.log("Déconnexion:", socket.id);
    for (const [code, room] of Object.entries(rooms)) {
      room.users = room.users.filter((id) => id !== socket.id);
      if (room.users.length === 0) {
        delete rooms[code];
      }
    }
  });
});

function generateCode() {
  return Math.random().toString(36).substring(2, 8).toUpperCase();
}

server.listen(3000, () => {
  console.log("Serveur SyncClip lancé sur le port 3000");
});---
title: Tutorial - Introduction
sidebar_label: Introduction
slug: introduction
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# Getting started

Welcome to the Socket.IO tutorial!

In this tutorial we'll create a basic chat application. It requires almost no basic prior knowledge of Node.JS or Socket.IO, so it’s ideal for users of all knowledge levels.

## Introduction

Writing a chat application with popular web applications stacks like LAMP (PHP) has normally been very hard. It involves polling the server for changes, keeping track of timestamps, and it’s a lot slower than it should be.

Sockets have traditionally been the solution around which most real-time chat systems are architected, providing a bi-directional communication channel between a client and a server.

This means that the server can *push* messages to clients. Whenever you write a chat message, the idea is that the server will get it and push it to all other connected clients.

## How to use this tutorial

### Tooling

Any text editor (from a basic text editor to a complete IDE such as [VS Code](https://code.visualstudio.com/)) should be sufficient to complete this tutorial.

Additionally, at the end of each step you will find a link to some online platforms ([CodeSandbox](https://codesandbox.io) and [StackBlitz](https://stackblitz.com), namely), allowing you to run the code directly from your browser:

![Screenshot of the CodeSandbox platform](/images/codesandbox.png)

### Syntax settings

In the Node.js world, there are two ways to import modules:

- the standard way: ECMAScript modules (or ESM)

```js
import { Server } from "socket.io";
```

Reference: https://nodejs.org/api/esm.html

- the legacy way: CommonJS

```js
const { Server } = require("socket.io");
```

Reference: https://nodejs.org/api/modules.html

Socket.IO supports both syntax. 

:::tip

We recommend using the ESM syntax in your project, though this might not always be feasible due to some packages not supporting this syntax.

:::

For your convenience, throughout the tutorial, each code block allows you to select your preferred syntax:

<Tabs groupId="lang">
  <TabItem value="cjs" label="CommonJS" default>

```js
const { Server } = require("socket.io");
```

  </TabItem>
  <TabItem value="mjs" label="ES modules">

```js
import { Server } from "socket.io";
```

  </TabItem>
</Tabs>


Ready? Click "Next" to get started.
