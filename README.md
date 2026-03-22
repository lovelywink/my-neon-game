# my-neon-game
一个测试用的霓虹像素设计游戏
[射击游戏.tml.txt](https://github.com/user-attachments/files/26163008/tml.txt)
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Neon Pixel Shooter - Gemini Edition</title>
    <style>
        :root {
            --neon-blue: #00f2fe;
            --neon-pink: #ff007c;
            --bg-dark: #020406;
        }

        body, html {
            margin: 0; padding: 0; width: 100%; height: 100%;
            background-color: var(--bg-dark);
            overflow: hidden; touch-action: none;
            font-family: 'Courier New', Courier, monospace;
        }

        /* Gemini 霓虹光晕背景 */
        .nebula-bg {
            position: absolute; top: 0; left: 0; width: 100%; height: 100%;
            z-index: -1; filter: blur(80px); opacity: 0.5;
            background: 
                radial-gradient(circle at 20% 30%, rgba(0, 242, 254, 0.4) 0%, transparent 50%),
                radial-gradient(circle at 80% 70%, rgba(112, 0, 255, 0.3) 0%, transparent 60%);
            animation: drift 15s infinite alternate ease-in-out;
        }

        @keyframes drift {
            from { transform: scale(1); }
            to { transform: scale(1.2) translate(20px, 20px); }
        }

        canvas { display: block; }

        /* UI 界面 */
        #ui {
            position: absolute; top: 20px; width: 100%;
            display: flex; justify-content: space-between;
            padding: 0 20px; box-sizing: border-box;
            color: white; pointer-events: none;
            text-shadow: 0 0 10px var(--neon-blue);
        }

        .score-board { font-size: 24px; font-weight: bold; }
        
        #game-over {
            position: absolute; top: 50%; left: 50%;
            transform: translate(-50%, -50%);
            text-align: center; color: white;
            display: none; background: rgba(0,0,0,0.8);
            padding: 40px; border-radius: 20px;
            border: 2px solid var(--neon-pink);
            box-shadow: 0 0 30px var(--neon-pink);
        }

        button {
            background: var(--neon-blue); border: none;
            padding: 10px 20px; font-size: 18px; cursor: pointer;
            margin-top: 20px; color: black; font-weight: bold;
            border-radius: 5px;
        }
    </style>
</head>
<body>

    <div class="nebula-bg"></div>

    <div id="ui">
        <div class="score-board">SCORE: <span id="score">0</span></div>
        <div style="opacity: 0.5;">v1.0</div>
    </div>

    <div id="game-over">
        <h1>GAME OVER</h1>
        <p>Your Score: <span id="final-score">0</span></p>
        <button onclick="resetGame()">RESTART</button>
    </div>

    <canvas id="gameCanvas"></canvas>

    <script>
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        const scoreEl = document.getElementById('score');
        const gameOverEl = document.getElementById('game-over');
        const finalScoreEl = document.getElementById('final-score');

        let score = 0;
        let gameActive = true;
        let player, bullets, enemies, particles;

        // 设置画布大小
        function resize() {
            canvas.width = window.innerWidth;
            canvas.height = window.innerHeight;
        }
        window.addEventListener('resize', resize);
        resize();

        // 玩家战机 (像素风)
        class Player {
            constructor() {
                this.width = 40;
                this.height = 40;
                this.x = canvas.width / 2;
                this.y = canvas.height - 80;
            }
            draw() {
                ctx.fillStyle = '#00f2fe';
                ctx.shadowBlur = 15;
                ctx.shadowColor = '#00f2fe';
                // 绘制像素飞机
                ctx.fillRect(this.x - 5, this.y, 10, 30);
                ctx.fillRect(this.x - 20, this.y + 15, 40, 10);
                ctx.shadowBlur = 0;
            }
        }

        // 子弹
        class Bullet {
            constructor(x, y) {
                this.x = x;
                this.y = y;
                this.speed = 10;
                this.radius = 3;
            }
            update() {
                this.y -= this.speed;
            }
            draw() {
                ctx.fillStyle = '#fff';
                ctx.beginPath();
                ctx.arc(this.x, this.y, this.radius, 0, Math.PI * 2);
                ctx.fill();
            }
        }

        // 敌人 (像素方块)
        class Enemy {
            constructor() {
                this.size = Math.random() * 30 + 20;
                this.x = Math.random() * (canvas.width - this.size);
                this.y = -this.size;
                this.speed = Math.random() * 2 + 2 + (score / 500);
                this.color = `hsl(${Math.random() * 360}, 70%, 60%)`;
            }
            update() {
                this.y += this.speed;
            }
            draw() {
                ctx.strokeStyle = this.color;
                ctx.lineWidth = 2;
                ctx.strokeRect(this.x, this.y, this.size, this.size);
                // 内部小方块
                ctx.fillStyle = this.color;
                ctx.fillRect(this.x + this.size/4, this.y + this.size/4, this.size/2, this.size/2);
            }
        }

        function init() {
            player = new Player();
            bullets = [];
            enemies = [];
            particles = [];
            score = 0;
            scoreEl.innerText = score;
            gameActive = true;
            gameOverEl.style.display = 'none';
        }

        // 触摸/鼠标移动
        window.addEventListener('mousemove', (e) => {
            player.x = e.clientX;
        });
        window.addEventListener('touchmove', (e) => {
            player.x = e.touches[0].clientX;
        });

        // 核心循环
        function spawnEnemies() {
            if (gameActive && Math.random() < 0.03) {
                enemies.push(new Enemy());
            }
        }

        function autoShoot() {
            if (gameActive && frame % 10 === 0) {
                bullets.push(new Bullet(player.x, player.y));
            }
        }

        let frame = 0;
        function animate() {
            if (!gameActive) return;
            requestAnimationFrame(animate);
            ctx.fillStyle = 'rgba(2, 4, 6, 0.3)'; // 拖尾效果
            ctx.fillRect(0, 0, canvas.width, canvas.height);

            frame++;
            player.draw();
            autoShoot();
            spawnEnemies();

            // 子弹处理
            bullets.forEach((bullet, index) => {
                bullet.update();
                bullet.draw();
                if (bullet.y < 0) bullets.splice(index, 1);
            });

            // 敌人处理
            enemies.forEach((enemy, eIndex) => {
                enemy.update();
                enemy.draw();

                // 碰撞检测：敌人撞到玩家
                if (enemy.y + enemy.size > player.y && 
                    enemy.x < player.x + 20 && 
                    enemy.x + enemy.size > player.x - 20) {
                    endGame();
                }

                // 碰撞检测：子弹打中敌人
                bullets.forEach((bullet, bIndex) => {
                    if (bullet.x > enemy.x && bullet.x < enemy.x + enemy.size &&
                        bullet.y > enemy.y && bullet.y < enemy.y + enemy.size) {
                        enemies.splice(eIndex, 1);
                        bullets.splice(bIndex, 1);
                        score += 10;
                        scoreEl.innerText = score;
                    }
                });

                if (enemy.y > canvas.height) enemies.splice(eIndex, 1);
            });
        }

        function endGame() {
            gameActive = false;
            gameOverEl.style.display = 'block';
            finalScoreEl.innerText = score;
        }

        function resetGame() {
            init();
            animate();
        }

        init();
        animate();
    </script>
</body>
</html>
