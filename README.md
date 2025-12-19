<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <title>AI 마피아 게임 - 6인용</title>
    <style>
        :root { --bg-color: #1a1a1a; --card-bg: #2d2d2d; --text: #e0e0e0; --accent: #e74c3c; }
        body { background-color: var(--bg-color); color: var(--text); font-family: 'Pretendard', sans-serif; display: flex; justify-content: center; padding: 20px; margin: 0; }
        .container { width: 900px; display: grid; grid-template-columns: 1fr 300px; gap: 20px; }
        
        /* 플레이어 리스트 */
        .player-grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 15px; }
        .player-card { background: var(--card-bg); padding: 15px; border-radius: 10px; text-align: center; border: 2px solid transparent; transition: 0.3s; position: relative; }
        .player-card.alive { cursor: pointer; }
        .player-card.dead { opacity: 0.5; background: #111; text-decoration: line-through; }
        .player-card.selected { border-color: var(--accent); transform: translateY(-5px); }
        .role-tag { font-size: 12px; padding: 2px 6px; border-radius: 4px; background: #444; margin-top: 5px; display: inline-block; }

        /* 로그 및 컨트롤 */
        .sidebar { display: flex; flex-direction: column; gap: 20px; }
        #log { background: #000; height: 400px; padding: 15px; border-radius: 10px; overflow-y: auto; font-size: 14px; line-height: 1.6; border: 1px solid #333; }
        .controls { background: var(--card-bg); padding: 20px; border-radius: 10px; text-align: center; }
        button { background: var(--accent); color: white; border: none; padding: 10px 20px; border-radius: 5px; cursor: pointer; font-weight: bold; width: 100%; }
        button:disabled { background: #555; cursor: not-allowed; }

        .phase-indicator { font-size: 24px; font-weight: bold; margin-bottom: 10px; color: var(--accent); text-align: center; }
        .night { color: #9b59b6; }
    </style>
</head>
<body>

<div class="container">
    <div class="main-panel">
        <div id="phase-text" class="phase-indicator">게임 시작 대기 중</div>
        <div class="player-grid" id="player-grid">
            </div>
    </div>

    <div class="sidebar">
        <div id="log"></div>
        <div class="controls">
            <div id="instruction" style="margin-bottom: 10px;">게임 시작 버튼을 누르세요.</div>
            <button id="action-btn" onclick="initGame()">게임 시작</button>
        </div>
    </div>
</div>

<script>
    const ROLES = { MAFIA: '마피아', POLICE: '경찰', DOCTOR: '의사', CITIZEN: '시민' };
    const PHASES = { READY: 0, NIGHT: 1, DAY_RESULTS: 2, DAY_VOTE: 3 };
    
    let players = [];
    let currentPhase = PHASES.READY;
    let myIndex = 0;
    let selectedPlayer = null;
    let dayCount = 1;

    const logEl = document.getElementById('log');
    const phaseEl = document.getElementById('phase-text');
    const instructionEl = document.getElementById('instruction');
    const actionBtn = document.getElementById('action-btn');

    function addLog(msg, color = '#fff') {
        const div = document.createElement('div');
        div.style.color = color;
        div.innerHTML = `> ${msg}`;
        logEl.appendChild(div);
        logEl.scrollTop = logEl.scrollHeight;
    }

    // 게임 초기화
    function initGame() {
        const rolePool = [ROLES.MAFIA, ROLES.POLICE, ROLES.DOCTOR, ROLES.CITIZEN, ROLES.CITIZEN, ROLES.CITIZEN];
        const shuffled = rolePool.sort(() => Math.random() - 0.5);
        
        players = shuffled.map((role, i) => ({
            id: i + 1,
            role: role,
            isAlive: true,
            isAI: i !== 0,
            name: i === 0 ? "나 (당신)" : `AI 플레이어 ${i + 1}`
        }));

        currentPhase = PHASES.READY;
        dayCount = 1;
        logEl.innerHTML = '';
        addLog("게임이 시작되었습니다! 당신의 직업은 <b>[" + players[myIndex].role + "]</b>입니다.", "#f1c40f");
        
        startNight();
    }

    function renderPlayers() {
        const grid = document.getElementById('player-grid');
        grid.innerHTML = '';
        players.forEach((p, i) => {
            const card = document.createElement('div');
            card.className = `player-card ${p.isAlive ? 'alive' : 'dead'} ${selectedPlayer === i ? 'selected' : ''}`;
            
            // 본인이거나 죽은 사람의 직업 공개 (경찰 조사 결과는 따로 처리)
            let roleInfo = (i === myIndex || !p.isAlive) ? `<div class="role-tag">${p.role}</div>` : '';
            
            card.innerHTML = `
                <div style="font-size: 40px">${p.isAlive ? '👤' : '💀'}</div>
                <div>${p.name}</div>
                ${roleInfo}
            `;
            
            if (p.isAlive && currentPhase !== PHASES.READY) {
                card.onclick = () => selectPlayer(i);
            }
            grid.appendChild(card);
        });
    }

    function selectPlayer(index) {
        if (!players[index].isAlive) return;
        selectedPlayer = index;
        renderPlayers();
    }

    // --- 밤 페이즈 ---
    function startNight() {
        currentPhase = PHASES.NIGHT;
        selectedPlayer = null;
        phaseEl.innerText = `제 ${dayCount}일 - 밤`;
        phaseEl.className = "phase-indicator night";
        renderPlayers();

        const myRole = players[myIndex].role;
        if (!players[myIndex].isAlive) {
            addLog("당신은 사망했습니다. 밤의 진행을 기다립니다.");
            setTimeout(processNight, 2000);
            return;
        }

        if (myRole === ROLES.MAFIA) instructionEl.innerText = "처단할 사람을 선택하세요.";
        else if (myRole === ROLES.DOCTOR) instructionEl.innerText = "살릴 사람을 선택하세요.";
        else if (myRole === ROLES.POLICE) instructionEl.innerText = "조사할 사람을 선택하세요.";
        else instructionEl.innerText = "마피아의 활동을 기다리는 중...";

        actionBtn.innerText = (myRole === ROLES.CITIZEN) ? "밤 넘기기" : "능력 사용";
        actionBtn.onclick = useAbility;
    }

    function useAbility() {
        if (players[myIndex].role !== ROLES.CITIZEN && selectedPlayer === null) {
            alert("대상을 선택해주세요!");
            return;
        }
        processNight();
    }

    function processNight() {
        // AI들의 행동 결정
        let mafiaTarget = -1;
        let doctorTarget = -1;
        let policeTarget = -1;

        players.forEach((p, i) => {
            if (!p.isAlive) return;
            if (p.isAI) {
                const targets = players.filter((target, idx) => target.isAlive && idx !== i);
                const randomTarget = players.indexOf(targets[Math.floor(Math.random() * targets.length)]);
                
                if (p.role === ROLES.MAFIA) mafiaTarget = randomTarget;
                if (p.role === ROLES.DOCTOR) doctorTarget = randomTarget;
                if (p.role === ROLES.POLICE) policeTarget = randomTarget;
            } else {
                // 플레이어의 선택 적용
                if (p.role === ROLES.MAFIA) mafiaTarget = selectedPlayer;
                if (p.role === ROLES.DOCTOR) doctorTarget = selectedPlayer;
                if (p.role === ROLES.POLICE) policeTarget = selectedPlayer;
            }
        });

        // 경찰 조사 결과 알림 (플레이어가 경찰일 때)
        if (players[myIndex].role === ROLES.POLICE && players[myIndex].isAlive) {
            const isMafia = players[policeTarget].role === ROLES.MAFIA;
            addLog(`조사 결과: ${players[policeTarget].name}은(는) <b>${isMafia ? '마피아입니다!' : '마피아가 아닙니다.'}</b>`, isMafia ? '#e74c3c' : '#3498db');
        }

        // 결과 계산
        let deadPlayer = -1;
        if (mafiaTarget !== -1 && mafiaTarget !== doctorTarget) {
            deadPlayer = mafiaTarget;
            players[deadPlayer].isAlive = false;
        }

        startDay(deadPlayer);
    }

    // --- 낮 페이즈 ---
    function startDay(deadIdx) {
        currentPhase = PHASES.DAY_RESULTS;
        phaseEl.innerText = `제 ${dayCount}일 - 낮`;
        phaseEl.className = "phase-indicator";
        
        addLog(`--- 제 ${dayCount}일 낮이 밝았습니다 ---`, "#2ecc71");
        if (deadIdx === -1) {
            addLog("지난밤에는 아무 일도 일어나지 않았습니다.");
        } else {
            addLog(`지난밤 <b>${players[deadIdx].name}</b>이(가) 살해당했습니다.`, "#e74c3c");
        }

        renderPlayers();
        if (checkGameOver()) return;

        instructionEl.innerText = "회의 후 투표를 진행하세요.";
        actionBtn.innerText = "투표 시작";
        actionBtn.onclick = startVoting;
    }

    function startVoting() {
        currentPhase = PHASES.DAY_VOTE;
        selectedPlayer = null;
        renderPlayers();
        addLog("투표가 시작되었습니다. 마피아로 의심되는 인물을 선택하세요.");
        instructionEl.innerText = "마피아를 골라 투표하세요.";
        actionBtn.innerText = "투표 완료";
        actionBtn.onclick = processVote;
    }

    function processVote() {
        if (players[myIndex].isAlive && selectedPlayer === null) {
            alert("투표할 대상을 선택하세요!");
            return;
        }

        // 투표 집계 (AI는 랜덤 투표)
        let votes = new Array(players.length).fill(0);
        
        players.forEach((p, i) => {
            if (!p.isAlive) return;
            let target;
            if (p.isAI) {
                const aliveOnes = players.map((p, idx) => p.isAlive ? idx : -1).filter(idx => idx !== -1);
                target = aliveOnes[Math.floor(Math.random() * aliveOnes.length)];
            } else {
                target = selectedPlayer;
            }
            votes[target]++;
        });

        // 최다 득표자 선출
        let maxVote = Math.max(...votes);
        let candidates = votes.map((v, i) => v === maxVote ? i : -1).filter(i => i !== -1);
        
        if (candidates.length > 1) {
            addLog("투표 결과 동점이 발생하여 아무도 처형되지 않았습니다.", "#95a5a6");
        } else {
            let executed = candidates[0];
            players[executed].isAlive = false;
            addLog(`투표 결과 <b>${players[executed].name}</b>이(가) 처형되었습니다.`, "#e67e22");
            addLog(`${players[executed].name}의 정체는 <b>[${players[executed].role}]</b>였습니다.`);
        }

        renderPlayers();
        if (checkGameOver()) return;

        dayCount++;
        actionBtn.innerText = "밤으로 넘어가기";
        actionBtn.onclick = startNight;
    }

    function checkGameOver() {
        const mafiaCount = players.filter(p => p.isAlive && p.role === ROLES.MAFIA).length;
        const citizenCount = players.filter(p => p.isAlive && p.role !== ROLES.MAFIA).length;

        if (mafiaCount === 0) {
            endGame("시민 승리! 모든 마피아가 소탕되었습니다.");
            return true;
        }
        if (mafiaCount >= citizenCount) {
            endGame("마피아 승리! 마피아가 도시를 장악했습니다.");
            return true;
        }
        return false;
    }

    function endGame(msg) {
        currentPhase = PHASES.READY;
        phaseEl.innerText = "게임 종료";
        addLog(`🏆 ${msg}`, "#f1c40f");
        instructionEl.innerText = "게임을 다시 시작하시겠습니까?";
        actionBtn.innerText = "다시 시작";
        actionBtn.onclick = initGame;
        
        // 결과 공개
        players.forEach(p => p.isAlive = false); // 카드 밝히기용 가짜 처리
        renderPlayers();
    }
</script>

</body>
</html>
    
