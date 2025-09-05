<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width,initial-scale=1">
  <title>Iblue Flappy Bird</title>
  <style>
    body { margin:0; background:#70c5ce; display:flex; align-items:center; justify-content:center; height:100vh; overflow:hidden; font-family:sans-serif; }
    canvas { background:#70c5ce; display:block; border:4px solid #0d6efd; border-radius:12px; box-shadow:0 6px 18px rgba(0,0,0,0.4); }
    #score { position:absolute; top:20px; left:50%; transform:translateX(-50%); font-size:28px; color:white; font-weight:bold; text-shadow:2px 2px 0 #000; }
    #gameOver { position:absolute; top:50%; left:50%; transform:translate(-50%, -50%); background:rgba(0,0,0,0.7); color:#fff; padding:20px 30px; border-radius:10px; display:none; text-align:center; }
    #gameOver button { margin-top:12px; padding:10px 20px; border:0; border-radius:8px; background:#0d6efd; color:#fff; font-size:16px; cursor:pointer; }
    #gameOver button:hover { background:#0954c7; }
  </style>
</head>
<body>
  <div id="score">0</div>
  <div id="gameOver">
    <h2>Game Over</h2>
    <p>Your Score: <span id="finalScore">0</span></p>
    <button onclick="restartGame()">Restart</button>
  </div>
  <canvas id="gameCanvas" width="360" height="512"></canvas>

  <script>
    const canvas = document.getElementById('gameCanvas');
    const ctx = canvas.getContext('2d');

    let frames = 0;
    let score = 0;
    let pipes = [];
    let gameOver = false;

    const bird = {
      x:50,
      y:150,
      w:34,
      h:24,
      gravity:0.25,
      lift:-4.6,
      velocity:0,
      draw(){
        ctx.fillStyle = "yellow";
        ctx.beginPath();
        ctx.arc(this.x, this.y, this.w/2, 0, Math.PI*2);
        ctx.fill();
      },
      update(){
        this.velocity += this.gravity;
        this.y += this.velocity;
        if(this.y + this.h/2 >= canvas.height){
          endGame();
        }
        if(this.y - this.h/2 <= 0){
          this.y = this.h/2;
          this.velocity = 0;
        }
      },
      flap(){
        this.velocity = this.lift;
      }
    };

    function drawPipes(){
      for(let i=0; i<pipes.length; i++){
        let p = pipes[i];
        ctx.fillStyle = "#228B22";
        ctx.fillRect(p.x, 0, p.w, p.top);
        ctx.fillRect(p.x, canvas.height - p.bottom, p.w, p.bottom);
      }
    }

    function updatePipes(){
      if(frames % 100 === 0){
        let gap = 100;
        let top = Math.random() * (canvas.height - gap - 100) + 20;
        let bottom = canvas.height - top - gap;
        pipes.push({ x:canvas.width, w:50, top:top, bottom:bottom });
      }
      for(let i=0; i<pipes.length; i++){
        let p = pipes[i];
        p.x -= 2;
        if(p.x + p.w < 0){
          pipes.splice(i,1);
          i--;
          score++;
          document.getElementById('score').innerText = score;
        }
        // collision
        if(bird.x + bird.w/2 > p.x && bird.x - bird.w/2 < p.x + p.w){
          if(bird.y - bird.h/2 < p.top || bird.y + bird.h/2 > canvas.height - p.bottom){
            endGame();
          }
        }
      }
    }

    function draw(){
      ctx.fillStyle = "#70c5ce";
      ctx.fillRect(0,0,canvas.width,canvas.height);
      bird.draw();
      drawPipes();
    }

    function update(){
      bird.update();
      updatePipes();
    }

    function loop(){
      if(!gameOver){
        update();
        draw();
        frames++;
        requestAnimationFrame(loop);
      }
    }

    function endGame(){
      gameOver = true;
      document.getElementById('finalScore').innerText = score;
      document.getElementById('gameOver').style.display = 'block';
    }

    function restartGame(){
      score = 0;
      frames = 0;
      pipes = [];
      bird.y = 150;
      bird.velocity = 0;
      gameOver = false;
      document.getElementById('score').innerText = score;
      document.getElementById('gameOver').style.display = 'none';
      loop();
    }

    document.addEventListener('keydown', e => {
      if(e.code === 'Space') bird.flap();
    });
    document.addEventListener('click', () => bird.flap());
    document.addEventListener('touchstart', () => bird.flap());

    loop();
  </script>
</body>
</html>
