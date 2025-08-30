# voicex-full-architecture
Полноценная архитектура для софта VoiceX


1) Детальное, пошаговое описание принципа работы (словами)

Коротко: система захватывает одновременно два потока сигналов — акустический микрофон и контактный/вибрационный датчик с гортани, синхронизирует их, извлекает набор признаков (частоты, амплитуда, pitch, jitter/shimmer, спектральные признаки), затем объединяет (fusion) и подаёт в модель DSP/ML, которая восстанавливает и «очищает» речь. На выходе — реконструированный сигнал с улучшенной разборчивостью и сохранением естественного тембра говорящего. Управление, конфигурация и обновления проходят через BLE → мобильное приложение.

Дальше — по шагам, максимально технически и конкретно.

Захват сигналов (input)
	1.	Датчики:
	•	Акустический микрофон (MIC) снимает звуковое поле вокруг говорящего (тембр, форманты, окружающий шум).
	•	Контактный датчик вибрации гортани (VIB, контактный микрофон/акселерометр) фиксирует механические колебания голосовых связок через кожу — этот канал устойчив к внешнему шуму и даёт «источник» голоса (glottal excitation).
	2.	АЦП и синхронизация:
	•	Оба канала дискретизируются ADC с контролируемой частотой (на практике: MIC ≥16 kHz, VIB ≥4–8 kHz). Важно: синхронизация по времени (timestamp/clock alignment) для корректного наложения признаков.

Предобработка (Preprocess)
	3.	Блоки предобработки:
	•	DC removal, предусиление, антиалиасинг.
	•	Буферизация в кадры (frames) — типично 20–40 ms окна с шагом 10–20 ms.
	•	Оконное взвешивание (Hanning/Hamming).
	4.	Синхронизация каналов:
	•	Оценка временной задержки между VIB и MIC (cross-correlation или GCC-PHAT) → выравнивание (delay compensation).
	•	При сильной асинхронии — корректировка сэмплитайма или компенсация фазы.

Выделение признаков (Feature extraction)
	5.	Акустические признаки (из MIC):
	•	STFT → спектрограммы (magnitude / log-magnitude).
	•	Mel-filterbank / MFCC.
	•	Formants (F1..Fn), спектральные моменты, энергия.
	6.	Глоттальные и вибрационные признаки (из VIB):
	•	Pitch / F0 (например, YIN/RobustPitch).
	•	Jitter / Shimmer (показатели нестабильности голосовой складки).
	•	Параметры glottal flow (через inverse filtering) — дают источник возбуждения.
	7.	Доп. признаки:
	•	delta / delta-delta, SNR-оценки, voice activity (VAD), шумовая метка.

Выравнивание и слияние (Alignment & Fusion)
	8.	Выравнивание в частотной/временной области — чтобы признаки были в одной системе координат.
	9.	Fusion strategies:
	•	Feature-level fusion: конкатенация вектор-признаков (MFCC_mic || features_vib) + нормировка;
	•	Attention/Weighting: динамический вес VIB vs MIC в зависимости от SNR; в шуме VIB получает больший вес;
	•	Decision-level fusion: независимые модели для MIC и VIB → объединение выходов (ensembling).

ML / DSP модель восстановления (Core)
	10.	Архитектуры (варианты):
	•	Спектрографическая U-Net / SE-UNet (map noisy magnitude → clean magnitude).
	•	CRNN / TCN / Conv-TasNet (end-to-end вейвформ-ориентированные).
	•	Гибрид: DSP-предобработка + легкая NN для коррекции (для MCU).
	11.	Задача обучения:
	•	На входе: (MIC features + VIB features), на выходе: чистый спектр или напрямую вейвформ.
	•	Потери: L1/L2 на спектре, perceptual losses (STOI/PESQ proxy), pitch-consistency loss.
	12.	Персонализация (он-устройство):
	•	Короткая калибровка: 30–60 с эталонной речи → вычисление speaker embedding (x-vector) или адаптивных параметров.
	•	Использование адаптационных слоёв (FiLM / adaptive gains) для подгонки модели под голос говорящего.
	13.	Реализация для MCU:
	•	Лёгкая/квантованная модель (int8), TFLite Micro или оптимизированный TCN, latency budget ≈ 20–80 ms.

Постобработка (Postprocess)
	14.	Реставрация фазы / синтез:
	•	Если модель производит magnitude spectrum → фазовая реконструкция (Griffin–Lim) или neural vocoder (WaveRNN/HiFiGAN) на мобильном устройстве для лучшего качества.
	15.	AGC / Limiter / Smoothing: выравнивание громкости, удаление клиппинга, сглаживание переходов.
	16.	Финальный выход: аудиосигнал выдаётся на динамик устройства или передаётся по BLE/USB/ Wi-Fi.

Управление, интерфейс и связь
	17.	BLE GATT (ble_communicator.py + protocols.py):
	•	Характеристики: MODE (Normal/Quiet/SOS), VOLUME, TELEMETRY (bat/RSSI/version), OTA, LOG.
	•	Команды из мобильного приложения меняют параметры pipeline (режимы, профиль, агрессивность шумоподавления).
	18.	Мобильное приложение:
	•	UI для режимов, калибровки профиля, записи образцов, локального/облачного обучения.
	•	Может выполнять вычислительно тяжёлый ML-инференс (neural vocoder) если устройство маломощное.
	19.	OTA (безопасно):
	•	Обновление прошивки через BLE/шарденные чанки, проверка подписи образа перед установкой.
	20.	Отладка / Wi-Fi сервер (wifi_api_server.py):
	•	Для разработки и логирования; отладочные данные и загрузка выборок для обучения.

Обучающий пайплайн (offline)
	21.	Сбор данных: синхронные записи VIB+MIC+TARGET (чистая речь) в разных условиях.
	22.	Аугментации: шумы, реверберация, pitch-shift.
	23.	Train/Val/Test split без перекрытия говорящих.
	24.	Метрики: STOI, PESQ, SI-SDR, WER (через ASR) и MOS.

Логика режима SOS / защита приватности
	25.	SOS: максимально агрессивная очистка, приоритет разборчивости, включён remote logging/telemetry (опционально).
	26.	Приватность: обработка и профили по умолчанию локально; передача на сервер — по явному согласию и в зашифрованном виде.

Как это отражено в коде (файлы)
	•	voicex_main.py — orchestrator: capture loop, вызов preprocess → features → infer → postprocess → output; обработка режимов, OTA и логики.
	•	ble_communicator.py — GATT server/client, приём команд, нотификации телеметрии.
	•	sensor.py — драйверы для VIB и MIC, ADC-read, первичная фильтрация.
	•	protocols.py — строковые/байтовые форматы команд, перечисления режимов и версии протокола.
	•	wifi_api_server.py — dev/debug API: экспорт логов, выгрузка записей, ручное тестирование.

# Схема архитектуры ввиде mermaid – flowchart TD

flowchart TD
  %% top-level: capture
  subgraph SOURCE["Источники сигнала"]
    A1[Акустический микрофон (MIC)\n16 kHz+] 
    A2[Контактный вибрационный датчик (VIB)\n4-8 kHz]
  end

  %% ADC and sync
  B1[ADC / Кодеки\nСинхронизация временных меток]
  A1 --> B1
  A2 --> B1

  %% Capture & Buffers
  C1[Capture & Buffers\nFrame: 20-40 ms / hop 10-20 ms]
  B1 --> C1

  %% Preprocess
  C2[Preprocess\nDC remove, pre-emphasis,\nwindowing]
  C1 --> C2

  %% Alignment
  C3[Временное выравнивание\n(Delay est. / GCC-PHAT / xcorr)]
  C2 --> C3

  %% Feature extraction
  subgraph FEAT["Feature Extraction"]
    F1[MIC: STFT -> Mel / MFCC\nformants, spectral moments]
    F2[VIB: Pitch (F0),\njitter/shimmer, glottal params]
    F3[VAD, energy, SNR-estimates]
  end
  C3 --> F1
  C3 --> F2
  C3 --> F3

  %% Fusion
  D1[Fusion Layer\nfeature-level concat OR\nattention-weighted fusion]
  F1 --> D1
  F2 --> D1
  F3 --> D1

  %% ML/DSP core
  E1[ML/DSP Core\n(denoise & voice restoration)\nU-Net / CRNN / TCN / lightweight model]
  D1 --> E1

  %% Personalization
  subgraph PERS["Персонализация"]
    P1[Calibration: 30-60s speech]
    P2[Speaker embedding (x-vector)\nFiLM/adaptive layers]
  end
  P1 --> P2
  P2 --> E1
  E1 --> PERS

  %% Postprocess and synthesis
  G1[Postprocess\nphase reconstruction (GL) / neural vocoder,\nAGC, limiter, smoothing]
  E1 --> G1

  %% Output
  H1[Audio output (device speaker) / Recording]
  G1 --> H1

  %% BLE & Mobile
  S1[BLE GATT (CHAR_MODE, CHAR_VOL,\nCHAR_TELEMETRY, CHAR_OTA, CHAR_LOG)]
  S2[Mobile App\n(UI, profiles, OTA, ML heavy vocoder)]
  S1 --- S2
  S1 -->|telemetry| T1[Telemetry DB / Logs]
  S1 -->|commands| C_ctrl[Control: MODE/VOLUME/OTA]
  C_ctrl --> voicem[voicex_main.py\napply_mode_settings]

  %% Interaction between device and mobile
  G1 -->|stream/preview| S1
  S2 -->|profile upload/ download| P1

  %% OTA path
  OTA1[OTA Server / Signed images]
  S2 --> OTA1
  OTA1 --> S1
  S1 -->|install| voicem

  %% Debug / Wi-Fi
  W1[Wi-Fi API Server (dev)\nwifi_api_server.py]
  voicem[voicex_main.py] -.-> W1
  W1 -.-> T1

  %% Sensors / drivers mapping
  subgraph HW_DRIVERS["Драйвера / firmware modules"]
    SD1[sensor.py - ADC drivers\npre-filter]
    SD2[ble_communicator.py - BLE stack]
    SD3[protocols.py - message formats]
    SD4[voicex_main.py - orchestrator]
  end
  SD1 --> voicem
  SD2 --> voicem
  SD3 --> SD2

  %% ML Training & Dataflow (offline)
  OFF1[Dataset: (VIB, MIC, TARGET)\naugmentation, train/val/test]
  OFF2[Training Pipeline\nloss = L1/L2 + perceptual + pitch-consistency]
  OFF1 --> OFF2
  OFF2 -->|model artifacts| MODEL[Quantized model (int8) / TFLite]
  MODEL --> E1

  %% Privacy & Security
  SEC1[Privacy: local processing by default\nExplicit consent for cloud]
  SEC2[Security: BLE encryption,\nOTA signature verification]
  SEC1 --- S1
  SEC2 --- OTA1

  %% Mode nodes and emergency
  MODE[Modes: NORMAL / QUIET / SOS]
  S2 --> MODE
  MODE --> voicem
  SOS[Emergency trigger (button/SOS)\nhigh-priority processing]
  SOS --> MODE


# VoiceX Architecture & Software

Автор: Таңатқан Асқар Ұланұлы, 2025, Казахстан  
Возраст: 16 лет  

## Авторские права
Весь исходный код, архитектура и документация проекта *VoiceX* принадлежат исключительно автору.  
Авторство подтверждается публикацией на GitHub и охраняется международным правом (Бернская конвенция).

## Лицензия
Этот проект распространяется под лицензией *Creative Commons Attribution-NonCommercial 4.0 International (CC BY-NC 4.0)*.  
- Разрешено: читать, изучать, использовать в некоммерческих целях.  
- Запрещено: коммерческое использование, модификация и распространение без письменного разрешения автора.  

## Патенты
Попытка патентования любых частей данного проекта третьими лицами будет считаться *плагиатом и нарушением авторских прав*.  
Публикация данного репозитория является *prior art*, исключающим возможность патентования.  

---
© Таңатқан Асқар Ұланұлы, 2025
