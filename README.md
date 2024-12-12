import serial
import csv
import pandas as pd
from urllib.request import urlopen
import gradio as gr
import threading
import openai

# 글로벌 변수 설정
SERIAL_PORT = '/dev/ttyUSB0'  # Arduino 포트 설정
BAUD_RATE = 9600
CSV_FILE = "dust_sensor_data.csv"
stop_measurement = False  # 센서 측정 중단 플래그
current_data = ""  # 현재 데이터를 저장하는 변수

# OpenAI API 키 설정
openai.api_key = "sk-your-openai-api-key"  # 자신의 OpenAI API 키로 대체하세요

# 센서 측정 시작 함수
def start_sensor():
    global stop_measurement, current_data
    stop_measurement = False

    def measure():
        global current_data
        with open(CSV_FILE, mode='w', newline='') as file:
            writer = csv.writer(file)
            writer.writerow(["Time (s)", "Dust Concentration (ug/m3)", "Air Quality"])
            try:
                with serial.Serial(SERIAL_PORT, BAUD_RATE, timeout=1) as ser:
                    while not stop_measurement:
                        line = ser.readline().decode('utf-8').strip()
                        if line and "," in line:
                            writer.writerow(line.split(","))
                            current_data = line  # 최신 데이터를 업데이트
            except Exception as e:
                print(f"Error: {e}")

    threading.Thread(target=measure, daemon=True).start()
    return "센서 측정을 시작합니다."

# 센서 측정 중단 함수
def stop_sensor():
    global stop_measurement
    stop_measurement = True
    return "센서 측정을 중단합니다."

# 실시간 데이터 출력 함수
def get_current_data():
    global current_data
    return f"현재 센서 데이터: {current_data}" if current_data else "현재 데이터가 없습니다."

# 기상청 API 실행 및 가장 최근값 반환 함수
def fetch_latest_api_value():
    url = 'https://apihub.kma.go.kr/api/typ01/url/kma_pm10.php?tm1=202412112020&tm2=202412122020&authKey=8diTRQm4TEKYk0UJuOxCsg'
    with urlopen(url) as f:
        html = f.read().decode('euc-kr')
    lines = html.split('\n')
    filtered_lines = [line for line in lines if ',   132,' in line]
    columns = ["TM", "STN", "PM10", "FLAG", " ", "MQC"]
    data = []
    for line in filtered_lines:
        values = line.split(',')
        data.append([value.strip() for value in values])
    df = pd.DataFrame(data, columns=columns[:len(data[0])])
    df.drop(['STN', 'FLAG', ' ', 'MQC'], axis=1, inplace=True)
    last_api_value = df['PM10'].iloc[-1]

    # Dust Sensor CSV에서 가장 최근 값 가져오기
    try:
        df_dust = pd.read_csv(CSV_FILE)
        last_sensor_value = df_dust['Dust Concentration (ug/m3)'].iloc[-1]
    except Exception as e:
        last_sensor_value = "센서 데이터 없음"

    return f"기상청 API 최근 값: {last_api_value}, 센서 최근 값: {last_sensor_value}"

# GPT 응답 생성 함수
def generate_gpt_response(user_message):
    try:
        response = openai.ChatCompletion.create(
            model="gpt-3.5-turbo",
            messages=[
                {"role": "system", "content": "You are a helpful assistant."},
                {"role": "user", "content": user_message}
            ]
        )
        return response['choices'][0]['message']['content']
    except Exception as e:
        return f"GPT 응답 생성 중 오류가 발생했습니다: {e}"

# 채팅 형식의 Gradio 인터페이스 구축
def process_chat(user_message, chat_history):
    global current_data

    if user_message.lower() == "센서 시작":
        response = start_sensor()
    elif user_message.lower() == "센서 중단":
        response = stop_sensor()
    elif user_message.lower() == "현재 데이터":
        response = get_current_data()
    elif user_message.lower() == "api 값":
        response = fetch_latest_api_value()
    else:
        # GPT 응답 생성
        response = generate_gpt_response(user_message)

    chat_history.append((user_message, response))
    return "", chat_history

with gr.Blocks() as demo:
    gr.Markdown("# Dust Sensor 및 기상청 API 통합 프로젝트 - 채팅형")

    chatbot = gr.Chatbot()
    user_input = gr.Textbox(label="명령어 또는 질문 입력")

    user_input.submit(process_chat, [user_input, chatbot], [user_input, chatbot])

demo.launch(share=True, debug=True)
