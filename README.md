# SpeechFeedback

End-to-End ASR (Automatic Speech Recognition) Feedback System

IPA 변환을 통하여 발음 그대로 인식하도록 하고 그에 대한 발음 피드백을 진행할 수 있도록 하는 것이 목표이다.

데이터셋 : [AIHub 한국인 대화음성](https://aihub.or.kr/aihubdata/data/view.do?currMenu=115&topMenu=100&aihubDataSe=realm&dataSetSn=130)

KoSpeech 툴킷 : [sooftware/kospeech](https://github.com/sooftware/kospeech)

IPA 변환기 : [표준발음 변환기](http://pronunciation.cs.pusan.ac.kr/)

<br/>

### Docker Image

KoSpeech (Using CUDA 12.0) : https://hub.docker.com/r/devtae/kospeech

1. `sudo docker run -it --gpus all --name devtae -v {하위 디렉토리}/한국인\ 대화\ 음성/Training/data/remote/PROJECT/AI학습데이터/KoreanSpeech/data:/workspace/data devtae/kospeech`

2. `sudo docker attach devtae`

<br/>

### How to done Preprocessing (IPA and Character Dictionary)

1. ipa_crawl.py 과 ipa_preprocess.py *(부산대학교 인공지능연구소의 허락을 받아야 실행할 수 있음 (중요))* 를 `한국인\ 대화\ 음성/Training/data/remote/PROJECT/AI학습데이터/KoreanSpeech/data` 에 넣는다.

2. `python3 ipa_preprocess.py` 를 실행하여 데이터에 대한 IPA 변환을 진행한다.

3. IPA 변환이 끝난 후, `KoSpeech/dataset/kspon/preprocess.sh` (해당 repo 에서 복사 및 붙여넣기 진행) 에서의 `DATASET_PATH` 에 `/workspace/data` 를 입력하고 `VOCAB_DEST` 에는 `/workspace/data/vocab` 를 입력한다.

4. `bash preprocess.sh` 를 통해 전처리를 완료한다.

5. 그 결과, `main.py` 가 있던 디렉토리에 `transcripts.txt` 가 생기고, 단어 사전은 설정된 `VOCAB_DEST` *(/workspace/data/vocab)* 폴더에 저장된다.

- 해당 레포에 있는 코드는 `1.Training` 에 대한 데이터를 모두 전처리하는 것이며, `2.Validation` 데이터를 이용하지 않는다. 따라서 별 다른 수정 없이 사용한다면, `1.Training` 에서 원하는 데이터들을 바탕으로 transcripts 를 형성시키고, 그 중에서도 일부를 떼어내 따로 evaluation 용 `transcripts_eval.txt` 파일을 만들어 사용하면 된다.

<br/>

### How to train `Deep Speech 2` model

1. `KoSpeech/configs/audio/fbank.yaml` *(melspectrogram.yaml, mfcc.yaml, spectrogram.yaml)* 에서 음원 확장명(.pcm or .wav)을 수정한다.

2. `KoSpeech/kospeech/data/data_loader.py` 에서 train, validation 데이터 수를 설정한다. (transcripts.txt 파일에서의 데이터 수)

3. main.py, eval.py, inference.py 에 대하여 단어 사전 경로를 `/workspace/data/vocab/aihub_labels.csv` 로 수정해준다.

4. `KoSpeech/configs/train/ds2_train.yaml` 에서 `transcripts_path: '/workspace/kospeech/dataset/kspon/transcripts.txt'` 로 설정한다.

5. 최종적으로, `python ./bin/main.py model=ds2 train=ds2_train train.dataset_path=/workspace/data` 를 실행한다.

<br/>

### How to solve the problem that occurs nan value of loss during training

- 다음의 코드 `/workspace/kospeech/kospeech/trainer/supervised_trainer.py` 에서 loss 값 계산 전에 nan 을 보정해주는 함수를 추가해준다.

#### Before

`loss = self.criterion(outputs.transpose(0, 1), targets[:, 1:], output_lengths, target_lengths)`

#### After

`loss = self.criterion(torch.nan_to_num(outputs).transpose(0, 1), targets[:, 1:], output_lengths, target_lengths) # Pytorch 1.8.0 부터 가능`

<br/>

### How to evaluate `Deep Speech 2` model

- 아래 코드를 바탕으로 평가를 진행한다.

- `python ./bin/eval.py eval.dataset_path=/workspace/data eval.transcripts_path=/workspace/kospeech/dataset/kspon/transcripts_eval.txt eval.model_path=/workspace/kospeech/outputs/{date}/{time}/model.pt`

<br/>

### How to inference the audio file using `Deep Speech 2` model

- 아래 코드를 바탕으로 해당 오디오 파일에 대하여 추론을 한다.

- `python3 ./bin/inference.py --model_path /workspace/kospeech/outputs/{date}/{time}/model.pt --audio_path /workspace/data/1.Training/2.원천데이터/1.방송/broadcast_01/001/broadcast_00000001.wav --device "cpu"`

