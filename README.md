const express = require('express');
const http = require('http');
const socketIo = require('socket.io');

const app = express();
const server = http.createServer(app);
const io = socketIo(server);

// Use environment port for hosting, or 3000 locally
const PORT = process.env.PORT || 3000;

app.use(express.static('public'));

const players = {};
const enemies = [];

io.on('connection', socket => {
  console.log('Player connected:', socket.id);

  players[socket.id] = {
    x: Math.random() * 10 - 5,
    z: Math.random() * 10 - 5,
    health: 100,
    score: 0
  };

  socket.emit('init', { players, enemies });
  socket.broadcast.emit('playerJoined', { id: socket.id, data: players[socket.id] });

  socket.on('move', data => {
    if (players[socket.id]) {
      players[socket.id].x = data.x;
      players[socket.id].z = data.z;
      socket.broadcast.emit('playerMoved', { id: socket.id, data: data });
    }
  });

  socket.on('shoot', bullet => {
    socket.broadcast.emit('playerShot', { id: socket.id, bullet });

    for (let id in players) {
      if (id !== socket.id) {
        const p = players[id];
        const dist = Math.hypot(bullet.x - p.x, bullet.z - p.z);
        if (dist < 1.5) {
          p.health -= 25;
          if (p.health <= 0) {
            players[socket.id].score += 10;
            io.to(id).emit('died');
            io.emit('playerScored', { id: socket.id, score: players[socket.id].score });
            
            p.health = 100;
            p.x = Math.random() * 10 - 5;
            p.z = Math.random() * 10 - 5;
            io.emit('playerMoved', { id: id, data: { x: p.x, z: p.z } });
          }
          io.emit('playerHealth', { id: id, health: p.health });
        }
      }
    }
  });

  socket.on('disconnect', () => {
    console.log('Player disconnected:', socket.id);
    delete players[socket.id];
    io.emit('playerLeft', socket.id);
  });
});

server.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
