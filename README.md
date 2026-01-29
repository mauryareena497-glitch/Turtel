<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Turtle Trek</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Fredoka:wght@400;600&display=swap');

        body {
            margin: 0;
            overflow: hidden;
            background-color: #006994;
            font-family: 'Fredoka', sans-serif;
            touch-action: none; /* Prevent scroll on mobile */
        }
        
        #gameCanvas {
            display: block;
            width: 100vw;
            height: 100vh;
        }

        .game-ui {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            pointer-events: none;
            display: flex;
            flex-direction: column;
            justify-content: center;
            align-items: center;
            text-align: center;
        }

        .interactive {
            pointer-events: auto;
        }

        .hidden {
            display: none !important;
        }

        /* Bubbles animation for background */
        .bubble {
            position: absolute;
            bottom: -50px;
            background-color: rgba(255, 255, 255, 0.2);
            border-radius: 50%;
            animation: rise 10s infinite ease-in;
            z-index: -1;
        }

        @keyframes rise {
            0% { bottom: -50px; transform: translateX(0); }
            50% { transform: translateX(50px); }
            100% { bottom: 110vh; transform: translateX(-50px); }
        }
    </style>
</head>
<body>

    <!-- Bubbles Container -->
    <div id="bubbles-container" class="absolute inset-0 overflow-hidden pointer-events-none z-0"></div>

    <canvas id="gameCanvas"></canvas>

    <!-- Score HUD -->
    <div id="scoreHud" class="absolute top-4 left-4 text-white text-2xl font-bold drop-shadow-md hidden z-10">
        Score: <span id="scoreVal">0</span>
    </div>
    
    <!-- Lives HUD -->
    <div id="livesHud" class="absolute top-4 right-4 text-white text-2xl font-bold drop-shadow-md hidden z-10">
        ❤️❤️❤️
    </div>

    <!-- Start Screen -->
    <div id="startScreen" class="game-ui z-20 bg-black/40 backdrop-blur-sm">
        <h1 class="text-6xl text-green-400 font-bold mb-4 drop-shadow-lg">Turtle Trek</h1>
        <p class="text-white text-xl mb-8 max-w-md">Help the turtle swim safely! Avoid the plastic trash and eat jellyfish.</p>
        
        <div class="bg-white/90 p-6 rounded-2xl shadow-xl max-w-sm interactive mx-4">
            <div class="flex gap-4 justify-center mb-6 text-gray-700">
                <div class="flex flex-col items-center">
                    <span class="text-3xl mb-2">⬆️⬇️</span>
                    <span class="text-xs font-bold">ARROWS</span>
                    <span class="text-xs">to move</span>
                </div>
                <div class="flex flex-col items-center justify-center font-bold text-gray-400">OR</div>
                <div class="flex flex-col items-center">
                    <span class="text-3xl mb-2">👆</span>
                    <span class="text-xs font-bold">TOUCH</span>
                    <span class="text-xs">and drag</span>
                </div>
            </div>
            <button id="startBtn" class="w-full bg-green-500 hover:bg-green-600 text-white font-bold py-4 rounded-xl text-xl transition transform hover:scale-105 active:scale-95 shadow-lg border-b-4 border-green-700">
                PLAY NOW 🐢
            </button>
        </div>
    </div>

    <!-- Game Over Screen -->
    <div id="gameOverScreen" class="game-ui hidden z-30 bg-black/60 backdrop-blur-md">
        <h2 class="text-5xl text-red-400 font-bold mb-2 drop-shadow-lg">Oh no!</h2>
        <p class="text-white text-xl mb-6">The ocean is dangerous.</p>
        
        <div class="bg-white p-8 rounded-2xl shadow-2xl interactive animate-bounce-in mx-4">
            <div class="text-gray-500 text-sm uppercase tracking-wide mb-1">Final Score</div>
            <div id="finalScore" class="text-5xl font-bold text-green-600 mb-6">0</div>
            
            <button id="restartBtn" class="w-full bg-blue-500 hover:bg-blue-600 text-white font-bold py-3 px-8 rounded-xl text-lg transition transform hover:scale-105 active:scale-95 shadow-lg border-b-4 border-blue-700">
                Try Again 🔄
            </button>
        </div>
    </div>

    <script>
        // --- Game Config & State ---
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        
        // Screens
        const startScreen = document.getElementById('startScreen');
        const gameOverScreen = document.getElementById('gameOverScreen');
        const scoreHud = document.getElementById('scoreHud');
        const livesHud = document.getElementById('livesHud');
        const scoreVal = document.getElementById('scoreVal');
        const finalScore = document.getElementById('finalScore');
        
        // Buttons
        const startBtn = document.getElementById('startBtn');
        const restartBtn = document.getElementById('restartBtn');

        let gameState = 'START'; // START, PLAYING, GAMEOVER
        let score = 0;
        let lives = 3;
        let gameSpeed = 3;
        let frameCount = 0;
        let lastTime = 0;
        
        // Assets
        const emojis = {
            player: '🐢',
            trash: ['🛍️', '🥤', '🥫', '🧴', '👞'],
            food: ['🪼', '🦐', '⭐'],
            bubbles: '🫧'
        };

        // --- Resizing ---
        function resize() {
            canvas.width = window.innerWidth;
            canvas.height = window.innerHeight;
        }
        window.addEventListener('resize', resize);
        resize();

        // --- Classes ---

        class Turtle {
            constructor() {
                this.width = 60;
                this.height = 40;
                this.x = 50;
                this.y = canvas.height / 2;
                this.vy = 0;
                this.speed = 5;
                this.targetY = null; // For touch control
            }

            update(input) {
                // Keyboard Movement
                if (input.keys.ArrowUp) this.y -= this.speed;
                if (input.keys.ArrowDown) this.y += this.speed;
                if (input.keys.ArrowLeft) this.x -= this.speed;
                if (input.keys.ArrowRight) this.x += this.speed;

                // Touch/Mouse Follow Movement
                if (input.isTouching && this.targetY !== null) {
                    const dy = this.targetY - this.y;
                    const dx = this.targetX - this.x;
                    
                    // Simple easing
                    this.y += dy * 0.1;
                    this.x += dx * 0.1;
                }

                // Boundaries
                if (this.y < 0) this.y = 0;
                if (this.y > canvas.height - this.height) this.y = canvas.height - this.height;
                if (this.x < 0) this.x = 0;
                if (this.x > canvas.width - this.width) this.x = canvas.width - this.width;
            }

            draw() {
                ctx.font = '50px Arial';
                ctx.textAlign = 'center';
                ctx.textBaseline = 'middle';
                // Flip turtle horizontally? No, default emoji faces left usually, let's just draw it.
                // Windows 🐢 faces left. Mac 🐢 faces right. 
                // Let's assume standard right-facing or neutral.
                ctx.save();
                ctx.translate(this.x + this.width/2, this.y + this.height/2);
                // Bobbing animation
                ctx.translate(0, Math.sin(Date.now() / 200) * 3);
                ctx.fillText(emojis.player, 0, 0);
                ctx.restore();
            }
            
            getBounds() {
                return {
                    x: this.x,
                    y: this.y,
                    w: this.width,
                    h: this.height
                };
            }
        }

        class Obstacle {
            constructor() {
                this.x = canvas.width + 50;
                this.y = Math.random() * (canvas.height - 50);
                this.size = 40 + Math.random() * 20;
                this.emoji = emojis.trash[Math.floor(Math.random() * emojis.trash.length)];
                this.speed = gameSpeed + Math.random() * 2;
                this.rotation = 0;
                this.rotationSpeed = (Math.random() - 0.5) * 0.05;
                this.markedForDeletion = false;
            }

            update() {
                this.x -= this.speed;
                this.rotation += this.rotationSpeed;
                if (this.x < -100) this.markedForDeletion = true;
            }

            draw() {
                ctx.save();
                ctx.translate(this.x, this.y);
                ctx.rotate(this.rotation);
                ctx.font = `${this.size}px Arial`;
                ctx.textAlign = 'center';
                ctx.textBaseline = 'middle';
                ctx.fillText(this.emoji, 0, 0);
                ctx.restore();
            }

            getBounds() {
                return { x: this.x - this.size/3, y: this.y - this.size/3, w: this.size/1.5, h: this.size/1.5 }; // Smaller hitbox than visual
            }
        }

        class Food {
            constructor() {
                this.x = canvas.width + 50;
                this.y = Math.random() * (canvas.height - 50);
                this.size = 35;
                this.emoji = emojis.food[Math.floor(Math.random() * emojis.food.length)];
                this.speed = gameSpeed;
                this.markedForDeletion = false;
                this.wobbleOffset = Math.random() * 100;
            }

            update() {
                this.x -= this.speed;
                this.y += Math.sin((this.x + this.wobbleOffset) * 0.05) * 1; // Wavy movement
                if (this.x < -100) this.markedForDeletion = true;
            }

            draw() {
                ctx.font = `${this.size}px Arial`;
                ctx.textAlign = 'center';
                ctx.textBaseline = 'middle';
                
                // Glow effect for food
                ctx.shadowColor = "#FFFF00";
                ctx.shadowBlur = 10;
                ctx.fillText(this.emoji, this.x, this.y);
                ctx.shadowBlur = 0; // Reset
            }

            getBounds() {
                return { x: this.x - this.size/2, y: this.y - this.size/2, w: this.size, h: this.size };
            }
        }

        class Particle {
            constructor(x, y, text) {
                this.x = x;
                this.y = y;
                this.text = text;
                this.vy = -1 - Math.random();
                this.alpha = 1;
                this.markedForDeletion = false;
            }
            update() {
                this.y += this.vy;
                this.alpha -= 0.02;
                if (this.alpha <= 0) this.markedForDeletion = true;
            }
            draw() {
                ctx.globalAlpha = this.alpha;
                ctx.font = '20px Arial';
                ctx.fillStyle = 'white';
                ctx.fillText(this.text, this.x, this.y);
                ctx.globalAlpha = 1;
            }
        }

        // --- Input Handling ---
        class InputHandler {
            constructor() {
                this.keys = {};
                this.isTouching = false;

                window.addEventListener('keydown', e => this.keys[e.code] = true);
                window.addEventListener('keyup', e => this.keys[e.code] = false);

                // Touch/Mouse
                const updateTouch = (e) => {
                    if (gameState !== 'PLAYING') return;
                    this.isTouching = true;
                    
                    let clientX, clientY;
                    if(e.touches && e.touches.length > 0) {
                        clientX = e.touches[0].clientX;
                        clientY = e.touches[0].clientY;
                    } else {
                        clientX = e.clientX;
                        clientY = e.clientY;
                    }

                    player.targetX = clientX - player.width/2;
                    player.targetY = clientY - player.height/2;
                };

                const endTouch = () => {
                    this.isTouching = false;
                    player.targetY = null;
                }

                window.addEventListener('mousedown', updateTouch);
                window.addEventListener('mousemove', e => { if(this.isTouching) updateTouch(e); });
                window.addEventListener('mouseup', endTouch);

                window.addEventListener('touchstart', (e) => {
                    // Prevent default only inside canvas to stop scrolling
                    if(e.target === canvas) e.preventDefault();
                    updateTouch(e);
                }, {passive: false});
                
                window.addEventListener('touchmove', (e) => {
                    if(e.target === canvas) e.preventDefault();
                    updateTouch(e);
                }, {passive: false});
                
                window.addEventListener('touchend', endTouch);
            }
        }

        // --- Game Logic ---
        
        let player;
        let obstacles = [];
        let foods = [];
        let particles = [];
        let input = new InputHandler();
        let backgroundBubbles = [];

        function initGame() {
            player = new Turtle();
            obstacles = [];
            foods = [];
            particles = [];
            score = 0;
            lives = 3;
            gameSpeed = 3;
            frameCount = 0;
            updateHud();
        }

        function checkCollision(rect1, rect2) {
            return (
                rect1.x < rect2.x + rect2.w &&
                rect1.x + rect1.w > rect2.x &&
                rect1.y < rect2.y + rect2.h &&
                rect1.y + rect1.h > rect2.y
            );
        }

        function updateHud() {
            scoreVal.innerText = score;
            let heartString = '';
            for(let i=0; i<lives; i++) heartString += '❤️';
            livesHud.innerText = heartString;
        }

        function spawnEntities() {
            frameCount++;
            
            // Speed up slightly over time
            if (frameCount % 600 === 0) gameSpeed += 0.2;

            // Spawn Obstacles
            // Becomes more frequent as speed increases
            let obstacleFreq = Math.max(40, 100 - gameSpeed * 5); 
            if (frameCount % Math.floor(obstacleFreq) === 0) {
                obstacles.push(new Obstacle());
            }

            // Spawn Food
            if (frameCount % 90 === 0) {
                foods.push(new Food());
            }
        }

        function drawBackground() {
            // Gradient
            let grad = ctx.createLinearGradient(0, 0, 0, canvas.height);
            grad.addColorStop(0, '#006994'); // Deep blue
            grad.addColorStop(1, '#009DC4'); // Lighter blue
            ctx.fillStyle = grad;
            ctx.fillRect(0, 0, canvas.width, canvas.height);

            // Floor
            ctx.fillStyle = '#C2B280'; // Sand
            ctx.fillRect(0, canvas.height - 30, canvas.width, 30);
            
            // Decor
            ctx.fillStyle = '#2E8B57'; // Seaweed color
            for(let i=0; i<canvas.width; i+= 100) {
                // Simple seaweed shapes
                ctx.beginPath();
                ctx.moveTo(i, canvas.height - 30);
                ctx.quadraticCurveTo(i + 10, canvas.height - 60, i, canvas.height - 90);
                ctx.lineTo(i + 5, canvas.height - 30);
                ctx.fill();
            }
        }

        function animate(timeStamp) {
            const deltaTime = timeStamp - lastTime;
            lastTime = timeStamp;

            if (gameState === 'PLAYING') {
                ctx.clearRect(0, 0, canvas.width, canvas.height);
                drawBackground();

                spawnEntities();

                // Player
                player.update(input);
                player.draw();
                
                const pBounds = player.getBounds();

                // Obstacles
                obstacles.forEach(obj => {
                    obj.update();
                    obj.draw();
                    
                    if (checkCollision(pBounds, obj.getBounds())) {
                        obj.markedForDeletion = true;
                        lives--;
                        particles.push(new Particle(player.x, player.y, "OUCH!"));
                        updateHud();
                        
                        // Shake effect
                        canvas.style.transform = `translate(${Math.random()*10-5}px, ${Math.random()*10-5}px)`;
                        setTimeout(() => canvas.style.transform = 'none', 200);

                        if (lives <= 0) {
                            endGame();
                        }
                    }
                });

                // Food
                foods.forEach(obj => {
                    obj.update();
                    obj.draw();

                    if (checkCollision(pBounds, obj.getBounds())) {
                        obj.markedForDeletion = true;
                        score += 10;
                        particles.push(new Particle(obj.x, obj.y, "+10"));
                        updateHud();
                    }
                });

                // Particles
                particles.forEach(p => {
                    p.update();
                    p.draw();
                });

                // Clean up arrays
                obstacles = obstacles.filter(o => !o.markedForDeletion);
                foods = foods.filter(f => !f.markedForDeletion);
                particles = particles.filter(p => !p.markedForDeletion);

                requestAnimationFrame(animate);
            }
        }

        function startGame() {
            gameState = 'PLAYING';
            startScreen.classList.add('hidden');
            gameOverScreen.classList.add('hidden');
            scoreHud.classList.remove('hidden');
            livesHud.classList.remove('hidden');
            initGame();
            lastTime = performance.now();
            requestAnimationFrame(animate);
        }

        function endGame() {
            gameState = 'GAMEOVER';
            scoreHud.classList.add('hidden');
            livesHud.classList.add('hidden');
            gameOverScreen.classList.remove('hidden');
            finalScore.innerText = score;
        }

        // --- Event Listeners ---
        startBtn.addEventListener('click', startGame);
        restartBtn.addEventListener('click', startGame);

        // --- CSS Bubbles ---
        function createCSSBubbles() {
            const container = document.getElementById('bubbles-container');
            const bubbleCount = 15;
            for(let i=0; i<bubbleCount; i++) {
                const b = document.createElement('div');
                b.className = 'bubble';
                b.style.left = Math.random() * 100 + 'vw';
                b.style.width = Math.random() * 20 + 10 + 'px';
                b.style.height = b.style.width;
                b.style.animationDuration = Math.random() * 5 + 5 + 's';
                b.style.animationDelay = Math.random() * 5 + 's';
                container.appendChild(b);
            }
        }
        createCSSBubbles();

        // Initial render for background
        drawBackground();
        ctx.fillStyle = "rgba(0,0,0,0.3)";
        ctx.fillRect(0,0,canvas.width, canvas.height); // Dim for start s
