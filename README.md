<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>インベーダーゲーム - ハイスピード版</title>
  <style>
    body {
      background-color: #111;
      color: #fff;
      font-family: monospace;
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
      min-height: 100vh;
      margin: 0;
    }
    h1 { margin: 10px 0; }
    #gameCanvas {
      background: #000;
      border: 2px solid #0f0;
      box-shadow: 0 0 20px rgba(0, 255, 0, 0.4);
    }
    .info {
      margin-top: 10px;
      font-size: 14px;
      color: #aaa;
    }
  </style>
</head>
<body>

  <h1>SPACE INVADERS - HIGH SPEED</h1>
  <canvas id="gameCanvas" width="480" height="540"></canvas>
  <div class="info">操作方法: [←][→] 移動 / [Space] 連射</div>

  <script>
    const canvas = document.getElementById('gameCanvas');
    const ctx = canvas.getContext('2d');

    // --- ゲーム設定と状態 ---
    let score = 0;
    let lives = 3;
    let gameOver = false;
    let gameClear = false;

    // キー入力状態
    const keys = { left: false, right: false, space: false };

    // ★ プレイヤー設定（速度 UP: 4 -> 7）
    const player = {
      x: canvas.width / 2 - 15,
      y: canvas.height - 40,
      width: 30,
      height: 15,
      speed: 7,
      invulnerable: 0
    };

    // ★ 弾設定（速度 UP: 7 -> 12, 最大連射数: 3）
    const playerBullets = [];
    const maxPlayerBullets = 3; 
    const enemyBullets = [];
    const bulletSpeed = 12;
    const enemyBulletSpeed = 6; // 敵弾も高速化: 3.5 -> 6
    let canShoot = true;

    // 撃破エフェクト（パーティクル）配列
    const particles = [];

    // ★ 敵の攻撃タイマー（攻撃頻度 UP: 60 -> 35）
    let enemyShootTimer = 0;
    const enemyShootInterval = 35;

    // 敵（インベーダー）設定
    const alienRows = 4;
    const alienCols = 8;
    const totalAliensCount = alienRows * alienCols;
    const alienWidth = 30;
    const alienHeight = 20;
    const alienPadding = 15;
    const alienOffsetTop = 50;
    const alienOffsetLeft = 35;
    
    // ★ 敵の基本速度（ベース速度 UP）
    const baseAlienSpeed = 2.0;
    let currentAlienSpeed = baseAlienSpeed;
    let alienDirection = 1;

    // 敵配列の初期化
    const aliens = [];
    for (let r = 0; r < alienRows; r++) {
      aliens[r] = [];
      for (let c = 0; c < alienCols; c++) {
        aliens[r][c] = {
          x: alienOffsetLeft + c * (alienWidth + alienPadding),
          y: alienOffsetTop + r * (alienHeight + alienPadding),
          alive: true,
          color: r % 2 === 0 ? '#ff007f' : '#00ffff'
        };
      }
    }

    // 防護壁（シェルター）の設定
    const shelterCount = 4;
    const shelterWidth = 44;
    const shelterHeight = 32;
    const shelterY = canvas.height - 110;
    const shelters = [];

    const blockCols = 4;
    const blockRows = 3;
    const blockW = shelterWidth / blockCols;
    const blockH = shelterHeight / blockRows;

    const totalSpacing = canvas.width - (shelterCount * shelterWidth);
    const shelterGap = totalSpacing / (shelterCount + 1);

    for (let i = 0; i < shelterCount; i++) {
      const startX = shelterGap + i * (shelterWidth + shelterGap);
      const blocks = [];
      for (let r = 0; r < blockRows; r++) {
        for (let c = 0; c < blockCols; c++) {
          if (r === blockRows - 1 && (c === 1 || c === 2)) continue;
          blocks.push({
            x: startX + c * blockW,
            y: shelterY + r * blockH,
            w: blockW,
            h: blockH,
            health: 3
          });
        }
      }
      shelters.push(blocks);
    }

    // --- キーボードイベント ---
    window.addEventListener('keydown', (e) => {
      if (e.code === 'ArrowLeft' || e.code === 'KeyA') keys.left = true;
      if (e.code === 'ArrowRight' || e.code === 'KeyD') keys.right = true;
      if (e.code === 'Space') {
        if (!keys.space && canShoot && !gameOver && !gameClear) {
          shootBullet();
        }
        keys.space = true;
      }
    });

    window.addEventListener('keyup', (e) => {
      if (e.code === 'ArrowLeft' || e.code === 'KeyA') keys.left = false;
      if (e.code === 'ArrowRight' || e.code === 'KeyD') keys.right = false;
      if (e.code === 'Space') {
        keys.space = false;
        canShoot = true;
      }
    });

    // ★ 弾の発射（画面内最大3発まで同時発射可能）
    function shootBullet() {
      if (playerBullets.length < maxPlayerBullets) {
        playerBullets.push({
          x: player.x + player.width / 2 - 2,
          y: player.y,
          width: 4,
          height: 12
        });
      }
      canShoot = false;
    }

    // 敵弾の発射
    function enemyShoot() {
      const bottomAliens = [];
      for (let c = 0; c < alienCols; c++) {
        for (let r = alienRows - 1; r >= 0; r--) {
          if (aliens[r][c].alive) {
            bottomAliens.push(aliens[r][c]);
            break;
          }
        }
      }

      if (bottomAliens.length > 0) {
        const shooter = bottomAliens[Math.floor(Math.random() * bottomAliens.length)];
        enemyBullets.push({
          x: shooter.x + alienWidth / 2 - 2,
          y: shooter.y + alienHeight,
          width: 4,
          height: 12
        });
      }
    }

    // ★【スピード強化】スピーディーで迫力のある撃破エフェクト
    function createExplosion(x, y, color) {
      const particleCount = 18; 
      for (let i = 0; i < particleCount; i++) {
        const angle = Math.random() * Math.PI * 2;
        const speed = Math.random() * 6 + 2; // 飛散スピード向上
        particles.push({
          x: x,
          y: y,
          dx: Math.cos(angle) * speed,
          dy: Math.sin(angle) * speed,
          size: Math.random() * 4 + 1,
          life: 25 + Math.random() * 15, // 素早く消滅
          maxLife: 40,
          color: color
        });
      }
    }

    function isColliding(rect1, rect2) {
      return (
        rect1.x < rect2.x + rect2.w &&
        rect1.x + rect1.width > rect2.x &&
        rect1.y < rect2.y + rect2.h &&
        rect1.y + rect1.height > rect2.y
      );
    }

    // --- 更新処理 ---
    function update() {
      if (gameOver || gameClear) return;

      if (player.invulnerable > 0) player.invulnerable--;

      // 長押し連射補助
      if (keys.space && canShoot) {
        shootBullet();
      }

      // 1. プレイヤー移動
      if (keys.left && player.x > 0) player.x -= player.speed;
      if (keys.right && player.x < canvas.width - player.width) player.x += player.speed;

      // 2. プレイヤー弾移動
      for (let i = playerBullets.length - 1; i >= 0; i--) {
        playerBullets[i].y -= bulletSpeed;
        if (playerBullets[i].y < 0) playerBullets.splice(i, 1);
      }

      // 3. 敵弾移動
      for (let i = enemyBullets.length - 1; i >= 0; i--) {
        enemyBullets[i].y += enemyBulletSpeed;
        if (enemyBullets[i].y > canvas.height) enemyBullets.splice(i, 1);
      }

      // 4. パーティクル移動
      for (let i = particles.length - 1; i >= 0; i--) {
        const p = particles[i];
        p.x += p.dx;
        p.y += p.dy;
        p.life--;
        p.size *= 0.94;

        if (p.life <= 0 || p.size < 0.5) {
          particles.splice(i, 1);
        }
      }

      // 5. 敵のタイマー攻撃判定
      enemyShootTimer++;
      if (enemyShootTimer >= enemyShootInterval) {
        enemyShoot();
        enemyShootTimer = 0;
      }

      // 6. ★ 敵の移動ロジック（残数に応じた段階的スピードアップ）
      let moveDown = false;
      let activeAliens = 0;

      for (let r = 0; r < alienRows; r++) {
        for (let c = 0; c < alienCols; c++) {
          if (aliens[r][c].alive) activeAliens++;
        }
      }

      if (activeAliens === 0) {
        gameClear = true;
        return;
      }

      // 敵が減るほど速度倍率がアップ（最大4倍速まで上昇）
      const speedMultiplier = 1 + (1 - activeAliens / totalAliensCount) * 3;
      currentAlienSpeed = baseAlienSpeed * speedMultiplier;

      // 壁チェック
      for (let r = 0; r < alienRows; r++) {
        for (let c = 0; c < alienCols; c++) {
          const a = aliens[r][c];
          if (a.alive) {
            if (
              (alienDirection === 1 && a.x + alienWidth >= canvas.width - 10) ||
              (alienDirection === -1 && a.x <= 10)
            ) {
              moveDown = true;
            }
          }
        }
      }

      if (moveDown) {
        alienDirection *= -1;
        for (let r = 0; r < alienRows; r++) {
          for (let c = 0; c < alienCols; c++) {
            aliens[r][c].y += 16; // 下降スピード UP: 12 -> 16
            if (aliens[r][c].alive && aliens[r][c].y + alienHeight >= player.y) {
              gameOver = true;
            }
          }
        }
      } else {
        for (let r = 0; r < alienRows; r++) {
          for (let c = 0; c < alienCols; c++) {
            aliens[r][c].x += currentAlienSpeed * alienDirection;
          }
        }
      }

      // 7. 衝突判定 (自機弾 vs 敵)
      playerBullets.forEach((pb, pbIndex) => {
        for (let r = 0; r < alienRows; r++) {
          for (let c = 0; c < alienCols; c++) {
            const a = aliens[r][c];
            if (a.alive) {
              if (
                pb.x < a.x + alienWidth &&
                pb.x + pb.width > a.x &&
                pb.y < a.y + alienHeight &&
                pb.y + pb.height > a.y
              ) {
                a.alive = false;
                createExplosion(a.x + alienWidth / 2, a.y + alienHeight / 2, a.color);
                playerBullets.splice(pbIndex, 1);
                score += 10;
                return;
              }
            }
          }
        }
      });

      // 8. 衝突判定 (弾 vs シェルター)
      playerBullets.forEach((pb, pbIndex) => {
        shelters.forEach(blocks => {
          blocks.forEach(block => {
            if (block.health > 0 && isColliding(pb, block)) {
              block.health--;
              createExplosion(pb.x, pb.y, '#00cc44');
              playerBullets.splice(pbIndex, 1);
            }
          });
        });
      });

      enemyBullets.forEach((eb, ebIndex) => {
        shelters.forEach(blocks => {
          blocks.forEach(block => {
            if (block.health > 0 && isColliding(eb, block)) {
              block.health--;
              createExplosion(eb.x, eb.y, '#ff3333');
              enemyBullets.splice(ebIndex, 1);
            }
          });
        });
      });

      // 9. 衝突判定 (敵弾 vs プレイヤー)
      if (player.invulnerable === 0) {
        enemyBullets.forEach((eb, ebIndex) => {
          if (
            eb.x < player.x + player.width &&
            eb.x + eb.width > player.x &&
            eb.y < player.y + player.height &&
            eb.y + eb.height > player.y
          ) {
            enemyBullets.splice(ebIndex, 1);
            createExplosion(player.x + player.width/2, player.y + player.height/2, '#0f0');
            lives--;
            player.invulnerable = 40; // 無敵時間短縮

            if (lives <= 0) {
              gameOver = true;
            }
          }
        });
      }
    }

    // --- 描画処理 ---
    function draw() {
      ctx.fillStyle = '#000';
      ctx.fillRect(0, 0, canvas.width, canvas.height);

      // プレイヤー描画
      if (player.invulnerable % 6 < 3) {
        ctx.fillStyle = '#0f0';
        ctx.fillRect(player.x, player.y, player.width, player.height);
        ctx.fillRect(player.x + 10, player.y - 6, 10, 6);
      }

      // 弾描画
      ctx.fillStyle = '#0f0';
      playerBullets.forEach(b => ctx.fillRect(b.x, b.y, b.width, b.height));

      ctx.fillStyle = '#ff3333';
      enemyBullets.forEach(b => ctx.fillRect(b.x, b.y, b.width, b.height));

      // パーティクル描画
      particles.forEach(p => {
        const alpha = p.life / p.maxLife;
        ctx.globalAlpha = alpha;
        ctx.fillStyle = p.color;
        ctx.beginPath();
        ctx.arc(p.x, p.y, p.size, 0, Math.PI * 2);
        ctx.fill();
      });
      ctx.globalAlpha = 1.0;

      // 敵描画
      for (let r = 0; r < alienRows; r++) {
        for (let c = 0; c < alienCols; c++) {
          const a = aliens[r][c];
          if (a.alive) {
            ctx.fillStyle = a.color;
            ctx.fillRect(a.x, a.y, alienWidth, alienHeight);
          }
        }
      }

      // シェルター描画
      shelters.forEach(blocks => {
        blocks.forEach(b => {
          if (b.health > 0) {
            ctx.fillStyle = b.health === 3 ? '#00cc44' : b.health === 2 ? '#66e088' : '#b3f0c6';
            ctx.fillRect(b.x, b.y, b.w, b.h);
          }
        });
      });

      // UI
      ctx.fillStyle = '#0f0';
      ctx.font = '16px monospace';
      ctx.textAlign = 'left';
      ctx.fillText(`SCORE: ${score}`, 15, 25);
      ctx.textAlign = 'right';
      ctx.fillText(`LIVES: ${'♥'.repeat(Math.max(0, lives))}`, canvas.width - 15, 25);

      if (gameOver) {
        ctx.fillStyle = '#f00';
        ctx.font = '36px monospace';
        ctx.textAlign = 'center';
        ctx.fillText('GAME OVER', canvas.width / 2, canvas.height / 2);
      } else if (gameClear) {
        ctx.fillStyle = '#0f0';
        ctx.font = '36px monospace';
        ctx.textAlign = 'center';
        ctx.fillText('MISSION CLEAR!', canvas.width / 2, canvas.height / 2);
      }
    }

    // --- メインループ ---
    function gameLoop() {
      update();
      draw();
      requestAnimationFrame(gameLoop);
    }

    gameLoop();
  </script>
</body>
</html>
