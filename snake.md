<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <title>貪吃蛇大逃殺：極速迴廊 - 第十組</title>
    <style>
        body {
            background-color: #000;
            color: #fff;
            display: flex;
            flex-direction: column;
            align-items: center;
            font-family: sans-serif;
            margin-top: 50px;
            user-select: none;
        }
        canvas {
            background-color: #141414;
            box-shadow: 0 0 20px rgba(0, 255, 255, 0.2);
            border-radius: 5px;
        }
        #instructions {
            margin-top: 15px;
            text-align: center;
            color: #aaa;
            line-height: 1.5;
        }
        span.key { background: #333; padding: 2px 6px; border-radius: 4px; color: #fff; }
    </style>
</head>
<body>

    <canvas id="gameCanvas" width="800" height="600"></canvas>
    
    <div id="instructions">
        操作說明：<span class="key">W</span> <span class="key">A</span> <span class="key">S</span> <span class="key">D</span> 或是 <span class="key">方向鍵</span> 移動<br>
        進階技巧：按 <span class="key">空白鍵</span> 衝刺 / 按 <span class="key">ESC</span> 暫停遊戲
    </div>

<script>
    // ==========================================
    // 1. 遊戲初始化與常數設定
    // ==========================================
    const canvas = document.getElementById('gameCanvas');
    const ctx = canvas.getContext('2d');

    const WIDTH = canvas.width;
    const HEIGHT = canvas.height;
    const GRID_SIZE = 20;
    const COLS = WIDTH / GRID_SIZE;
    const ROWS = HEIGHT / GRID_SIZE;

    // 遊戲狀態 (新增 STATE_PAUSED 暫停狀態)
    const STATE_MENU = 0, STATE_PLAYING = 1, STATE_GAMEOVER = 2, STATE_WIN = 3, STATE_PAUSED = 4;
    let state = STATE_MENU;

    // 顏色定義
    const COLORS = {
        grid: '#282828',
        green: '#00c800', lightGreen: '#32ff32',
        red: '#c80000', lightRed: '#ff3232',
        yellow: '#ffd700', cyan: '#00ffff',
        blue: '#3264ff', shieldAura: '#64c8ff',
        white: '#ffffff', gray: '#646464', hoverGray: '#969696'
    };

    // 遊戲變數
    let playerSnake, playerDir, enemySnakes, food, shieldItem, score, exitPos;
    let enemyMoveCounter, hasShield, invincibleTimer, dashSpeedFrames;
    let lastMoveTime = 0;
    const targetScore = 5;

    // 輸入控制
    let mousePos = { x: 0, y: 0 };
    let mouseClicked = false;
    let nextDir = { x: 1, y: 0 }; 

    const mod = (n, m) => ((n % m) + m) % m;
    const isSame = (a, b) => a[0] === b[0] && a[1] === b[1];
    const contains = (arr, node) => arr.some(n => isSame(n, node));

    // ==========================================
    // 2. 貪婪追擊 AI：直線追擊
    // ==========================================
    function getGreedyNextStep(start, target) {
        let [x, y] = start;
        let [tx, ty] = target;
        let dx = 0, dy = 0;
        
        let distX = tx - x;
        if (Math.abs(distX) > COLS / 2) distX = distX > 0 ? distX - COLS : distX + COLS;
        
        let distY = ty - y;
        if (Math.abs(distY) > ROWS / 2) distY = distY > 0 ? distY - ROWS : distY + ROWS;

        if (Math.abs(distX) > Math.abs(distY)) {
            dx = distX > 0 ? 1 : -1;
        } else if (Math.abs(distY) > 0) {
            dy = distY > 0 ? 1 : -1;
        } else
