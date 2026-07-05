<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <title>Fourier Epicycles Art</title>
    <style>
        body {
            margin: 0;
            background-color: #111;
            color: #fff;
            font-family: sans-serif;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            height: 100vh;
            overflow: hidden;
        }
        canvas {
            border: 1px solid #333;
            background-color: #000;
            cursor: crosshair;
        }
        #controls {
            margin-top: 10px;
        }
        button {
            padding: 8px 16px;
            background-color: #333;
            color: white;
            border: none;
            border-radius: 4px;
            cursor: pointer;
        }
        button:hover { background-color: #444; }
    </style>
</head>
<body>

    <h3>画面をドラッグして一筆書きで絵を描いてね</h3>
    <canvas id="canvas" width="600" height="600"></canvas>
    <div id="controls">
        <button id="clearBtn">クリア / 描き直す</button>
    </div>

<script>
const canvas = document.getElementById('canvas');
const ctx = canvas.getContext('2d');
const clearBtn = document.getElementById('clearBtn');

let rawPoints = []; // ユーザーが描いた生データ [{x, y}, ...]
let fourierX = [];  // DFT結果
let time = 0;       // アニメーションの時間軸
let path = [];      // フーリエが再描画した軌跡の保存用
let isDrawing = false;
let mode = 'drawing'; // 'drawing' または 'playing'

// --- 離散フーリエ変換 (DFT) ---
// 2次元座標を複素数として扱い、各周波数の振幅と位相を求める
function dft(points) {
    const N = points.length;
    const X = [];
    
    // 計算負荷軽減のため、最大周波数を制限（サンプリング数が多い場合は間引くなどが必要）
    for (let k = 0; k < N; k++) {
        let re = 0;
        let im = 0;
        for (let n = 0; n < N; n++) {
            const phi = (Math.PI * 2 * k * n) / N;
            re += points[n].x * Math.cos(phi) + points[n].y * Math.sin(phi);
            im += -points[n].x * Math.sin(phi) + points[n].y * Math.cos(phi);
        }
        re = re / N;
        im = im / N;

        const freq = k;
        const amp = Math.sqrt(re * re + im * im);
        const phase = Math.atan2(im, re);

        X.push({ freq, amp, phase });
    }
    // 振幅の大きい順にソートすると、アニメーションが見栄え良くなります
    return X.sort((a, b) => b.amp - a.amp);
}

// --- お絵描きイベント ---
canvas.addEventListener('mousedown', (e) => {
    if (mode !== 'drawing') return;
    isDrawing = true;
    rawPoints = [];
    path = [];
    time = 0;
    const rect = canvas.getBoundingClientRect();
    rawPoints.push({ x: e.clientX - rect.left, y: e.clientY - rect.top });
});

canvas.addEventListener('mousemove', (e) => {
    if (!isDrawing) return;
    const rect = canvas.getBoundingClientRect();
    const x = e.clientX - rect.left;
    const y = e.clientY - rect.top;
    
    // 前の点と同じ座標ならスキップ
    if (rawPoints.length > 0) {
        const last = rawPoints[rawPoints.length - 1];
        if (last.x === x && last.y === y) return;
    }
    
    rawPoints.push({ x, y });

    // 生の線をリアルタイムに描画
    ctx.strokeStyle = '#555';
    ctx.lineWidth = 2;
    ctx.beginPath();
    ctx.moveTo(rawPoints[rawPoints.length-2].x, rawPoints[rawPoints.length-2].y);
    ctx.lineTo(x, y);
    ctx.stroke();
});

window.addEventListener('mouseup', () => {
    if (!isDrawing) return;
    isDrawing = false;
    if (rawPoints.length > 2) {
        // データが多すぎると重くなるため、最大500点程度に間引く（簡易実装）
        if (rawPoints.length > 500) {
            const step = Math.ceil(rawPoints.length / 500);
            rawPoints = rawPoints.filter((_, i) => i % step === 0);
        }
        fourierX = dft(rawPoints);
        mode = 'playing';
        animate();
    }
});

// --- 周転円（エピサイクル）の描画と計算 ---
function drawEpicycles(x, y, fourier) {
    for (let i = 0; i < fourier.length; i++) {
        const prevX = x;
        const prevY = y;
        const { freq, amp, phase } = fourier[i];
        
        // 座標の更新 (合成波の計算)
        x += amp * Math.cos(freq * time + phase);
        y += amp * Math.sin(freq * time + phase);

        // 円の描画（振幅が小さすぎるものは省略して描画負荷を下げる）
        if (amp > 0.5) {
            ctx.strokeStyle = 'rgba(255, 255, 255, 0.08)';
            ctx.beginPath();
            ctx.arc(prevX, prevY, amp, 0, Math.PI * 2);
            ctx.stroke();

            // 半径の線の描画
            ctx.strokeStyle = 'rgba(0, 150, 255, 0.4)';
            ctx.beginPath();
            ctx.moveTo(prevX, prevY);
            ctx.lineTo(x, y);
            ctx.stroke();
        }
    }
    return { x, y };
}

// --- アニメーションループ ---
function animate() {
    if (mode !== 'playing') return;

    ctx.clearRect(0, 0, canvas.width, canvas.height);

    // 円を連結して回し、その先端の座標（v）を取得
    const v = drawEpicycles(canvas.width / 2, canvas.height / 2, fourierX);
    
    // 再現された軌跡を配列の先頭に追加
    path.unshift(v);

    // 再現されたアートの線を描画
    ctx.strokeStyle = '#ff007f';
    ctx.lineWidth = 2.5;
    ctx.beginPath();
    for (let i = 0; i < path.length; i++) {
        if (i === 0) ctx.moveTo(path[i].x, path[i].y);
        else ctx.lineTo(path[i].x, path[i].y);
    }
    ctx.stroke();

    // 1周期（2π）終わったら軌跡の古い部分をカット、またはループ
    const dt = (Math.PI * 2) / fourierX.length;
    time += dt;

    if (time > Math.PI * 2) {
        time = 0;
        path = []; // ループ時に軌跡をリセット（お好みで残してもOK）
    }

    requestAnimationFrame(animate);
}

// クリアボタン
clearBtn.addEventListener('click', () => {
    mode = 'drawing';
    rawPoints = [];
    path = [];
    time = 0;
    ctx.clearRect(0, 0, canvas.width, canvas.height);
});
</script>
</body>
</html>
