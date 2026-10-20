# 왼쪽 ACT v2 — 5만·10만 스텝 로컬 평가

갱신: 2026-10-20. 다운로드한 두 모델은 모두 `chun-hwan/omx-unload-left-v2`로 학습한 **왼쪽 v2**다. 오른쪽 모델과 기존 v1 모델은 별도로 관리한다. 파일 정리·설정 읽기는 확인했고, 이 정리 작업에서 실제 팔 평가는 실행하지 않았다.

## 1. 모델 경로와 검증 결과

| 모델 | STEP_PAD | 로컬 경로 |
| --- | --- | --- |
| 왼쪽 v2 5만 | 050000 | `/home/team2/omx_models/omx_left_act_v2/050000/pretrained_model` |
| 왼쪽 v2 10만 | 100000 | `/home/team2/omx_models/omx_left_act_v2/100000/pretrained_model` |

각 폴더에 설정·모델·전처리·후처리·정규화 파일 7개를 함께 넣었다. ZIP 압축 해제·JSON·safetensors 형식·프로세서의 상대 파일 참조·ACT 설정 읽기를 확인했다. 모델 가중치는 234개 텐서, 전처리·후처리 정규화 가중치는 각각 90개 텐서다. 두 모델의 가중치는 서로 다르고 같은 스텝의 v1 가중치와도 다르다. 원본 ZIP은 `/home/team2/Downloads`에 유지한다.

입력은 관절 상태 6개와 `front`·`wrist` RGB 영상 각각 640×480, 출력은 관절 동작 6개다. 두 모델의 `chunk_size=100`, `n_action_steps=100`이다. 모델 파일 자체와 학습 메타데이터는 수정하지 않았다. 파일명에 붙은 스텝 값을 기준으로 폴더를 구분했으며 가중치 파일 자체가 학습 스텝을 증명하는 것은 아니다.

모델·녹화 데이터는 Git 저장소 밖에 있고 문서만 Git으로 전달한다. 다른 컴퓨터에서 평가하려면 해당 `pretrained_model` 폴더의 7개 파일도 함께 옮긴다.

## 2. 먼저 한 에피소드 확인

카메라 미리보기·원격조종·다른 평가를 종료하고, 수집 때와 같은 카메라 구도·카트·왼쪽 바구니·대기 자세를 준비한다. 물건 하나를 옮기는 동작부터 감독하며 확인한다. 아래 블록은 **실제 팔이 정책대로 움직이는 평가 명령**이다.

맨 위 `STEP_PAD` 한 곳을 `050000` 또는 `100000`으로 바꾸고 전체 블록을 복사한다. 먼저 5만을 확인한 다음 같은 물건·배치를 복원해 10만과 비교한다. 10만이 더 좋다고 미리 가정하지 않는다.

```bash
source /home/team2/venv/il/bin/activate
cd /home/team2/il_ws/src/lerobot

mkdir -p /home/team2/omx_record_logs
set -o pipefail

STEP_PAD="050000"  # 5만: 050000 / 10만: 100000
EVAL_TAG=$(date +%Y%m%d_%H%M%S)

lerobot-record \
  --robot.type=omx_follower \
  --robot.port=/dev/serial/by-id/usb-ROBOTIS_OpenRB-150_1B8E02E050323952582E3120FF071F2E-if00 \
  --robot.id=omx_follower_arm \
  --robot.cameras='{ front: {type: opencv, index_or_path: "/dev/v4l/by-id/usb-HFSD-231214-J_SNAP_U2-video-index0", width: 640, height: 480, fps: 30, fourcc: MJPG}, wrist: {type: opencv, index_or_path: "/dev/v4l/by-id/usb-Innomaker_Innomaker-U20CAM-720P_SN0001-video-index0", width: 640, height: 480, fps: 30, fourcc: MJPG}}' \
  --policy.path="/home/team2/omx_models/omx_left_act_v2/${STEP_PAD}/pretrained_model" \
  --policy.device=cuda \
  --display_data=true \
  --dataset.repo_id="chun-hwan/eval_omx_left_v2_s${STEP_PAD}_single_${EVAL_TAG}" \
  --dataset.single_task="Move one item from the cart to the left basket and return to the waiting pose." \
  --dataset.fps=30 \
  --dataset.num_episodes=1 \
  --dataset.episode_time_s=30 \
  --dataset.reset_time_s=0 \
  --dataset.num_image_writer_processes=0 \
  --dataset.num_image_writer_threads_per_camera=4 \
  --dataset.streaming_encoding=false \
  --dataset.push_to_hub=false \
  --dataset.private=true 2>&1 | tee "/home/team2/omx_record_logs/eval_left_v2_s${STEP_PAD}_single_${EVAL_TAG}.log"
```

한 에피소드는 최대 30초다. 동작 확인 후 `Esc`로 일찍 종료할 수 있다. 물건 이동 완료나 빈 카트를 자동으로 감지해 정지하는 명령은 아니다. 평가 데이터셋 이름과 로그에 스텝·실행 시각이 포함되며 HF 업로드는 끈다.

## 3. 카트 3개 자동 평가 — 화살표 없이 진행

단일 동작을 확인한 뒤 아래 블록을 쓴다. 남은 8종 중 물건 3개를 싣고, 모델마다 같은 시작 배치를 준비한다. `STEP_PAD` 한 곳만 바꾸면 된다.

**30초 실행 → 저장 → 다음 30초 실행 → 저장 → 마지막 30초 실행 후 종료**한다. 중간 준비 시간은 0초라 방향키를 누를 필요가 없다. 약 90초에 로딩·저장·인코딩 시간이 더해진다. 에피소드마다 정책 상태가 초기화되며, 한 번도 멈추지 않는 90초 단일 동작과는 다르다.

```bash
source /home/team2/venv/il/bin/activate
cd /home/team2/il_ws/src/lerobot

mkdir -p /home/team2/omx_record_logs
set -o pipefail

STEP_PAD="050000"  # 5만: 050000 / 10만: 100000
EVAL_TAG=$(date +%Y%m%d_%H%M%S)

lerobot-record \
  --robot.type=omx_follower \
  --robot.port=/dev/serial/by-id/usb-ROBOTIS_OpenRB-150_1B8E02E050323952582E3120FF071F2E-if00 \
  --robot.id=omx_follower_arm \
  --robot.cameras='{ front: {type: opencv, index_or_path: "/dev/v4l/by-id/usb-HFSD-231214-J_SNAP_U2-video-index0", width: 640, height: 480, fps: 30, fourcc: MJPG}, wrist: {type: opencv, index_or_path: "/dev/v4l/by-id/usb-Innomaker_Innomaker-U20CAM-720P_SN0001-video-index0", width: 640, height: 480, fps: 30, fourcc: MJPG}}' \
  --policy.path="/home/team2/omx_models/omx_left_act_v2/${STEP_PAD}/pretrained_model" \
  --policy.device=cuda \
  --display_data=true \
  --dataset.repo_id="chun-hwan/eval_omx_left_v2_s${STEP_PAD}_cart_${EVAL_TAG}" \
  --dataset.single_task="Move one item from the cart to the left basket and return to the waiting pose." \
  --dataset.fps=30 \
  --dataset.num_episodes=3 \
  --dataset.episode_time_s=30 \
  --dataset.reset_time_s=0 \
  --dataset.num_image_writer_processes=0 \
  --dataset.num_image_writer_threads_per_camera=4 \
  --dataset.streaming_encoding=false \
  --dataset.push_to_hub=false \
  --dataset.private=true 2>&1 | tee "/home/team2/omx_record_logs/eval_left_v2_s${STEP_PAD}_cart_${EVAL_TAG}.log"
```

세 에피소드 실행이 세 개 성공을 뜻하지 않는다. 한 에피소드에서 실제로 옮긴 수가 다를 수 있고, 실패해 물건이 남아도 시간에 따라 다음 에피소드로 넘어간다. 실제 담긴 물건·복귀·남은 물건을 관찰한다. 카트가 먼저 비거나 중단이 필요하면 `Esc`로 종료한다. 성공 수는 실제 영상과 결과로 판정한다.

## 4. 결과 기록

평가 영상은 `/home/team2/.cache/huggingface/lerobot/chun-hwan/eval_omx_left_v2_s<스텝>_<single 또는 cart>_<실행시각>/`에 저장된다. 실행 로그는 `/home/team2/omx_record_logs/`에 있다.

| 모델 | 물건·배치 | 정상 담기 /3 | 대기 자세 복귀 | 주요 실패·개입 | 평가 데이터셋 |
| --- | --- | --- | --- | --- | --- |
| v2 5만 | 미기록 | — | — | — | — |
| v2 10만 | 미기록 | — | — | — | — |

공중에서 닫기·헛집기·운반 중 낙하·바구니 밖 놓기·복귀 실패를 구분해 기록한다. 새 수집 영상은 [LEFT_RERECORD.md](LEFT_RERECORD.md), 오른쪽 수집·학습은 [RIGHT_RERECORD.md](RIGHT_RERECORD.md)를 참고한다.
