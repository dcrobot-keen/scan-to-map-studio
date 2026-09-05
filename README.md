> **이 저장소는 2026-09-06 에 `dcrobot-keen/pathfinder`(Fleet Studio 모노레포)의 `scan-engine/` 으로 합쳐졌습니다.**
> 이후 개발은 그쪽에서 합니다(`git subtree` 로 히스토리 보존). 이 저장소는 아카이브용으로만 남습니다.

# scan-to-map-studio

iPhone LiDAR 실내 스캔 → 천장 제거 베이스맵 자동 생성 → 로봇 occupancy grid와 2D 정합(registration)까지의 파이프라인. 설계 배경과 의사결정 근거는 [PLAN.md](PLAN.md) 참고.

**보안 제약**: 회사 로봇(SPOT / ROS2 SLAM 스택) 데이터는 반출 불가 — 이 코드만 내부망으로 가져가서, 거기서 실제 로봇 지도로 검증하는 구조. 아래 체크리스트는 **회사 가기 전에 미리** 확인해서, 가서 막히는 일이 없게 하기 위한 것.

## 회사 가기 전 사전 체크리스트

- [ ] **회사 내부망에서 `pip install -r requirements.txt`가 될지 확인** — PyPI 접근이 막혀있는 사내망이면 여기서 막힘. 안 되면 지금(외부망에서) 아래로 wheel을 미리 받아서 USB 등으로 함께 가져갈 것:
      ```
      pip download -r requirements.txt -d vendor_wheels
      ```
      회사에서는 `pip install --no-index --find-links vendor_wheels -r requirements.txt`로 설치.
- [ ] **회사 내부망에서 `git clone git@github.com:dcrobot-keen/scan-to-map-studio.git`이 될지 확인** — GitHub 자체가 막혀있거나 SSH 키가 그 PC에 없으면 안 됨. 안 되면 USB로 이 폴더를 통째로 복사(`.git` 포함)하는 걸로 대체.
- [ ] **로봇 PC에 Python 3.9 이상이 있는지** — numpy/scipy/plyfile/matplotlib은 3.9~3.13 전 구간에서 동작. 회사 PC의 파이썬 버전을 미리 알아두면 좋음.
- [ ] **로봇 쪽에서 `ros2 run nav2_map_server map_saver_cli` 명령이 있는지** — nav2 기반이면 보통 있음. 없으면 "지도 내보내기" 부분만 그 로봇의 SLAM 스택에 맞게 대체 확인 필요.

## 빠른 시작

```
pip install -r requirements.txt
python tests/test_preprocess.py      # 천장 제거 파이프라인 자체 검증
python tests/test_rasterize.py       # occupancy grid 래스터화 자체 검증
python tests/test_registration.py    # ICP 정합 자체 검증
```
세 개 다 `PASS`가 뜨면 코드 자체는 정상 — 이제 실제 데이터로 넘어가면 됨.

## 회사에서 할 일 (실제 로봇 지도로 검증)

### 방법 A — 한 번에 (권장)

```
ros2 run nav2_map_server map_saver_cli -f robot_map   # 로봇 쪽에서 지도 내보내기

python scripts/studio.py new officescan
python scripts/studio.py process officescan --usdz <scan.usdz> --robot-map robot_map
```
`projects/officescan/report.html`을 열면 원본→베이스맵→2D지도→정합결과→뷰어 링크까지 한 페이지에 다 나옴 (`--robot-map` 생략하면 정합 단계만 건너뜀). 리포트의 `align.html` 링크에서 정합 결과를 눈으로 확인/조정하고, 최종 값을 `python scripts/export_tf.py`에 넘기면 로봇 쪽에 등록할 ROS tf 명령어가 나옴 — 자세한 이유는 방법 B의 3~4단계 참고.

### 방법 B — 단계별로 (디버깅용)

1. **로봇 쪽에서 지도 내보내기**
   ```
   ros2 run nav2_map_server map_saver_cli -f robot_map
   ```
   → `robot_map.pgm` + `robot_map.yaml` 생성

2. **베이스맵 준비**
   ```
   python scripts/usdz_to_ply.py <scan.usdz> raw.ply
   python scripts/remove_ceiling.py raw.ply base_map.ply
   python scripts/rasterize_base_map.py base_map.ply base_map/map --png
   ```

3. **정합 실행 + 눈으로 확인/조정**
   ```
   python scripts/register_maps.py base_map/map robot_map --png overlay.png --align-html align.html
   ```
   → 콘솔에 자동 ICP 회전각/이동량/RMSE 출력, `overlay.png`에 두 지도가 겹쳐진 그림 저장.
   **`align.html`을 브라우저로 열어서 반드시 눈으로 확인할 것** — 실내 지도는 복도·방이 반복되는 구조라 ICP가 국소적으로 그럴듯하지만 전역적으로 틀린 정렬에 빠질 수 있고, RMSE/겹침 수치만으로는 이걸 못 잡아냄 (실제 사례: 회전 12도가 정답인데 ICP가 -4도로 잘못 수렴했는데 겹침%는 93%로 높게 나왔음). 화면에서 두 지도 윤곽이 어긋나 보이면 드래그(이동) / Shift+드래그(로봇맵 제자리 회전)로 직접 맞추고, "변환값 내보내기" 버튼으로 최종 회전/이동을 `registration_transform.json`으로 저장.

4. **로봇 쪽에 좌표변환 전달 (ROS tf)**
   ```
   python scripts/export_tf.py registration_transform.json
   ```
   → `ros2 run tf2_ros static_transform_publisher ...` 명령어와 `launch.py` `Node()` 스니펫을 둘 다 출력. 베이스맵을 통째로 로봇 좌표계로 재투영해서 넘기는 대신, `frame_id=scan_basemap`(이 베이스맵) ↔ `child_frame_id=map`(로봇 자신의 SLAM 프레임)으로 고정 tf 하나만 등록해두는 방식 — 로봇 쪽 nav2/RViz가 별도 파싱 코드 없이 표준 tf 조회만으로 바로 두 좌표계를 연결해서 씀. `align.html`에서 "ROS tf 명령어 보기" 버튼으로 같은 걸 바로 확인할 수도 있음.

방법 A가 실패하면 방법 B로 어느 단계에서 막히는지 좁혀보는 용도로 사용.

## 그 외 도구

- `python scripts/usdz_to_ply.py scan.usdz scan.ply` — iPhone 스캔 앱이 내보낸 .usdz를 PLY로 변환 (`usd-core` 필요)
- `python scripts/build_overlay_viewer.py <base_map_prefix> viewer.html [--trajectory traj.json]` — 베이스맵+로봇 궤적을 하나의 self-contained HTML로 만들어 재생(타임라인 스크러버) — 서버/네트워크 불필요, 더블클릭으로 바로 열림
- `python scripts/build_gltf_overlay.py --mesh scan.usdz --points base_map.ply:255,0,0 --output overlay.glb` — 원본 스캔(텍스처 메시)과 처리된 포인트클라우드를 하나의 glb로 합쳐서 [gltf-inspector](https://github.com/dcrobot-keen/gltf-inspector)로 3D 확인 (`trimesh`, `pygltflib`, `Pillow` 필요). gltf-inspector는 파일 하나만 불러오는 구조라 미리 합쳐야 함 — 자세한 내용은 PLAN.md의 "3D 오버레이 뷰어" 절 참고.

## 문제 생기면

- `pip install` 실패(오프라인) → 위 체크리스트의 `vendor_wheels` 방법 사용
- `register_maps.py`가 `robot_map.yaml`을 못 읽음 → 실제 로 나온 yaml 내용과 에러 메시지를 그대로 기록해두면, 그 사내망 nav2 배포판 차이에 맞춰 `studio/rasterize.py`의 로더를 보강할 수 있음
- 그 외 막히는 지점은 PLAN.md §6(미해결 질문)·§7-1(검증 가이드)에 이미 알려진 한계가 정리되어 있으니 먼저 확인

## 정합 워크스페이스 서버 (여러 스캔을 한 좌표계로)

iPhone 앱의 프로젝트 zip(스캔 여러 개 + `group_alignment.json`)을 폴더에 풀어 두고 서버를 띄우면,
브라우저에서 스캔을 끌어 맞추고 저장하는 것만으로 합성 slicemap이 다시 만들어져 시뮬레이터
`worlds/`로 내보내진다. 세부 설계는 전략 문서(아티팩트 "스캔 정합 워크스페이스") 참고.

```powershell
# 그룹 폴더(zip을 푼 곳)와 내보낼 곳(ros-chromium 시뮬레이터 worlds/)을 지정해 서버 실행
$env:STUDIO_GROUPS_DIR = 'D:\code\robot-project\vps-system\data'
$env:STUDIO_PUBLISH_DIR = 'D:\code\robot-project\ros-chromium\simulator\worlds'
.venv\Scripts\python.exe -m uvicorn server.app:app --port 8000
```

- `http://localhost:8000/groups` — 그룹 목록. 그룹을 열면 스캔별 슬라이스가 없을 때 처음 한 번 만든다(스캔당 수십 초).
- 워크스페이스에서 드래그·회전·기준점 쌍·ICP 마무리(겹침이 1.5 m 이상일 때만 열림)로 맞추고
  "서버에 저장 → 합성 반영"을 누르면 `group_alignment.json`이 갱신되고 `merged.slicemap.json`/`.png`가
  그룹 폴더에, `<그룹>.slicemap.json`이 `STUDIO_PUBLISH_DIR`에 쓰인다.
- 시뮬레이터는 slicemap 파일을 월드로 바로 읽는다: `SIM_WORLD=worlds/<그룹>.slicemap.json`.
  nav.html의 iPhone map에도 같은 파일을 넣으면 된다.
- 서버 없이 쓰려면 `scripts/align_workspace.py`(오프라인 페이지, 저장은 다운로드) +
  `scripts/merge_slicemaps.py`.

API: `GET /api/groups`, `GET /api/groups/{g}`, `POST /api/groups/{g}/prepare`,
`GET|PUT /api/groups/{g}/alignment`, `POST /api/groups/{g}/icp`, `GET /api/groups/{g}/merged.png`.

### 바닥 이미지 오버레이 (앱 floorplan.png)

앱(ios-capture)이 스캔마다 내보내는 `floorplan.png/json`(5 cm/px 톱다운 바닥 텍스처 + 궤적,
format_version 2)을 `studio/floorplan.py`가 읽어

- **워크스페이스**: 각 스캔의 슬라이스 아래에 같은 정합 변환으로 겹쳐 그립니다("바닥 이미지" 체크박스와
  불투명도 슬라이더). 벽 셀만 보고 맞추는 대신 실제 바닥 무늬로 정합을 확인할 수 있습니다. 앱의 회색
  배경(픽셀 (0,0) 색)은 투명 처리합니다.
- **저장 시**: 모든 스캔의 바닥 이미지를 `group_alignment.json`으로 합쳐 `merged.floor.png/.json`
  (merged 슬라이스맵과 같은 격자: 같은 origin/해상도/크기, `floor-image-v1`)을 만들고, publish 디렉터리에
  `<group>.floor.png/.json`으로 함께 내보냅니다. pathfinder는 이 파일을 `POST /api/projects/from-slicemap`의
  `floor`로 받아 프로젝트 배경으로 깝니다(`scripts/create-project-from-slicemap.mjs`가 옆 파일을 자동으로 붙임).
- `GET /api/groups/<name>/merged.floor.png`로 합성 결과를 볼 수 있습니다.

픽셀 규약(FloorPlanRenderer.swift v2): `x = origin_x + col*res`, `z = origin_top_z + row*res`(row 0 = 최소 z),
슬라이스 평면에서는 `y = -z`라 이미지 위쪽이 +y 입니다. 테스트: `python tests/test_floorplan.py`.
