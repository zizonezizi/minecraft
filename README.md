<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Simple Web Minecraft Prototype</title>
    <style>
        body {
            margin: 0;
            overflow: hidden; /* 스크롤바 제거 */
        }
        #crosshair {
            position: absolute;
            top: 50%;
            left: 50%;
            width: 20px;
            height: 20px;
            background-color: transparent;
            border: 2px solid white;
            border-radius: 50%;
            transform: translate(-50%, -50%);
            pointer-events: none; /* 마우스 이벤트가 통과하도록 설정 */
            z-index: 100;
        }
        #instructions {
            position: absolute;
            top: 10px;
            left: 10px;
            color: white;
            background: rgba(0,0,0,0.5);
            padding: 10px;
            border-radius: 5px;
            font-family: monospace;
            pointer-events: none;
        }
    </style>
    <script type="importmap">
        {
            "imports": {
                "three": "https://unpkg.com/three@0.160.0/build/three.module.js",
                "three/addons/": "https://unpkg.com/three@0.160.0/examples/jsm/"
            }
        }
    </script>
</head>
<body>
    <div id="crosshair"></div>
    <div id="instructions">
        클릭하여 게임 시작<br>
        이동: W, A, S, D<br>
        위/아래: Space / Shift<br>
        블록 설치: 우클릭<br>
        블록 파괴: 좌클릭<br>
        시선 이동: 마우스<br>
        ESC: 마우스 잠금 해제
    </div>

    <script type="module">
        import * as THREE from 'three';
        import { PointerLockControls } from 'three/addons/controls/PointerLockControls.js';

        let camera, scene, renderer, controls;
        let raycaster;
        const objects = []; // 블록들을 담을 배열

        let moveForward = false;
        let moveBackward = false;
        let moveLeft = false;
        let moveRight = false;
        let moveUp = false;
        let moveDown = false;

        const velocity = new THREE.Vector3();
        const direction = new THREE.Vector3();

        // 블록 재질 및 지오메트리 공통 사용 (최적화)
        const boxGeometry = new THREE.BoxGeometry(1, 1, 1);
        // 잔디 블록 색상 (초록색)
        const grassMaterial = new THREE.MeshLambertMaterial({ color: 0x55aa55 });
        // 흙 블록 색상 (갈색 - 우클릭으로 설치 시)
        const dirtMaterial = new THREE.MeshLambertMaterial({ color: 0x885533 });

        init();
        animate();

        function init() {
            // 1. 씬 설정
            scene = new THREE.Scene();
            scene.background = new THREE.Color(0x87ceeb); // 하늘색 배경
            scene.fog = new THREE.Fog(0x87ceeb, 10, 50); // 안개 효과

            // 2. 카메라 설정
            camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
            camera.position.y = 2; // 플레이어 눈높이

            // 3. 조명 설정
            const ambientLight = new THREE.AmbientLight(0xcccccc); // 환경광
            scene.add(ambientLight);

            const directionalLight = new THREE.DirectionalLight(0xffffff, 1.5); // 직사광 (태양)
            directionalLight.position.set(1, 1, 0.5).normalize();
            scene.add(directionalLight);

            // 4. 렌더러 설정
            renderer = new THREE.WebGLRenderer({ antialias: true });
            renderer.setPixelRatio(window.devicePixelRatio);
            renderer.setSize(window.innerWidth, window.innerHeight);
            document.body.appendChild(renderer.domElement);

            // 5. 컨트롤 설정 (1인칭 시점)
            controls = new PointerLockControls(camera, document.body);
            
            const instructions = document.getElementById('instructions');

            controls.addEventListener('lock', function () {
                instructions.style.display = 'none';
            });

            controls.addEventListener('unlock', function () {
                instructions.style.display = '';
            });

            // 화면 클릭 시 포인터 잠금 요청
            document.addEventListener('click', function () {
               if(!controls.isLocked) controls.lock();
            }, false);

            // 6. 초기 바닥 생성
            createFloor();

            // 7. 레이캐스터 설정 (블록 상호작용 용)
            raycaster = new THREE.Raycaster();
            raycaster.far = 10; // 너무 먼 곳은 클릭 안 되게 설정

            // 이벤트 리스너 등록
            document.addEventListener('keydown', onKeyDown);
            document.addEventListener('keyup', onKeyUp);
            document.addEventListener('mousedown', onMouseDown);
            window.addEventListener('resize', onWindowResize);
        }

        function createFloor() {
            const floorSize = 20;
            for (let x = -floorSize / 2; x < floorSize / 2; x++) {
                for (let z = -floorSize / 2; z < floorSize / 2; z++) {
                    const voxel = new THREE.Mesh(boxGeometry, grassMaterial);
                    voxel.position.set(x, 0, z);
                    scene.add(voxel);
                    objects.push(voxel);
                }
            }
        }

        function onWindowResize() {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        }

        function onKeyDown(event) {
            switch (event.code) {
                case 'KeyW': moveForward = true; break;
                case 'KeyA': moveLeft = true; break;
                case 'KeyS': moveBackward = true; break;
                case 'KeyD': moveRight = true; break;
                case 'Space': moveUp = true; break;
                case 'ShiftLeft': moveDown = true; break;
            }
        }

        function onKeyUp(event) {
            switch (event.code) {
                case 'KeyW': moveForward = false; break;
                case 'KeyA': moveLeft = false; break;
                case 'KeyS': moveBackward = false; break;
                case 'KeyD': moveRight = false; break;
                case 'Space': moveUp = false; break;
                case 'ShiftLeft': moveDown = false; break;
            }
        }

        // 마우스 클릭 핸들러 (블록 설치/파괴)
        function onMouseDown(event) {
            if (!controls.isLocked) return;

            // 화면 중앙에서 레이저 발사
            raycaster.setFromCamera(new THREE.Vector2(), camera);
            const intersects = raycaster.intersectObjects(objects, false);

            if (intersects.length > 0) {
                const intersect = intersects[0];

                // 좌클릭 (버튼 0): 블록 파괴
                if (event.button === 0) {
                    if (intersect.object.position.y > 0) { // 바닥(y=0)은 파괴 못하게 막음 (선택사항)
                        scene.remove(intersect.object);
                        objects.splice(objects.indexOf(intersect.object), 1);
                    }
                }
                // 우클릭 (버튼 2): 블록 설치
                else if (event.button === 2) {
                    const voxel = new THREE.Mesh(boxGeometry, dirtMaterial);
                    // 클릭한 지점의 면 법선 벡터를 이용해 새 블록 위치 계산
                    voxel.position.copy(intersect.point).add(intersect.face.normal);
                    voxel.position.divideScalar(1).floor().multiplyScalar(1).addScalar(0.5);
                    
                    scene.add(voxel);
                    objects.push(voxel);
                }
            }
        }


        function animate() {
            requestAnimationFrame(animate);

            const delta = 0.015; // 프레임 간 시간 간격 (단순화)

            if (controls.isLocked === true) {
                // 이동 속도 및 감속 설정
                velocity.x -= velocity.x * 10.0 * delta;
                velocity.z -= velocity.z * 10.0 * delta;
                velocity.y -= velocity.y * 10.0 * delta;

                direction.z = Number(moveForward) - Number(moveBackward);
                direction.x = Number(moveRight) - Number(moveLeft);
                direction.y = Number(moveUp) - Number(moveDown);
                direction.normalize(); // 대각선 이동 속도 일정하게

                const speed = 10.0;
                if (moveForward || moveBackward) velocity.z -= direction.z * speed * delta;
                if (moveLeft || moveRight) velocity.x -= direction.x * speed * delta;
                if (moveUp || moveDown) velocity.y += direction.y * speed * delta;

                controls.moveRight(-velocity.x * delta);
                controls.moveForward(-velocity.z * delta);
                camera.position.y += velocity.y * delta;
            }

            renderer.render(scene, camera);
        }
    </script>
</body>
</html>
