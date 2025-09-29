import serial
import time
import tkinter as tk
from gpiozero import OutputDevice
from threading import Thread
import csv
import matplotlib.pyplot as plt
from matplotlib.animation import FuncAnimation
from datetime import datetime, timedelta

# ---------------------------- 설정 ----------------------------
SERIAL_PORT = '/dev/ttyACM0'
BAUDRATE = 9600
RELAY_PIN = 18
LOG_FILE = "weight_log.csv"

# ---------------------------- GPIO 초기화 ----------------------------
relay = OutputDevice(RELAY_PIN, active_high=True, initial_value=False)

# ---------------------------- 데이터 변수 ----------------------------
sec1_values = []
sec10_avg = 0
values_1min = []
avg_1min = 0
saturation_weight = None
vwc_values = []
time_values = []
relay_events = []  # (time, "ON"/"OFF")
saturation_set = False

relay_state = False
relay_on_start_time = None
relay_off_time = None
update_saturation_flag = False
allow_relay = True  # 새로운 saturation weight 갱신 전까지 relay ON 제한

# ---------------------------- 시리얼 연결 ----------------------------
ser = serial.Serial(SERIAL_PORT, BAUDRATE, timeout=2)
time.sleep(2)

# ---------------------------- GUI ----------------------------
root = tk.Tk()
root.title("로드셀 기반 자동관수시스템")

# 입력창
tk.Label(root, text="포화 VWC (%)").pack()
entry_saturation_vwc = tk.Entry(root)
entry_saturation_vwc.pack()

tk.Label(root, text="관수개시 VWC (%)").pack()
entry_threshold_vwc = tk.Entry(root)
entry_threshold_vwc.pack()

tk.Label(root, text="트레이 용적 (mL)").pack()
entry_tray_volume = tk.Entry(root)
entry_tray_volume.pack()

tk.Label(root, text="관수 시간 (s)").pack()
entry_relay_duration = tk.Entry(root)
entry_relay_duration.pack()

# 상태 라벨
sec10_label = tk.Label(root, text="10초 평균: 0.00 g")
sec10_label.pack()
min_label = tk.Label(root, text="1분 평균: 0.00 g")
min_label.pack()
vwc_label = tk.Label(root, text="VWC: 0.000 %")
vwc_label.pack()
status_label = tk.Label(root, text="상태: 대기 중")
status_label.pack()
sat_label = tk.Label(root, text="포화 무게: 설정 안됨")
sat_label.pack()
weight_drop_label = tk.Label(root, text="무게 감소 기준점: 계산 안됨")
weight_drop_label.pack()

# ---------------------------- 버튼 & 카운트다운 ----------------------------
tare_after_id = None
cal_after_id = None

def send_command(cmd):
    ser.write((cmd + '\n').encode())

def countdown(label, base_text, seconds, kind):
    global tare_after_id, cal_after_id
    if seconds >= 0:
        label.config(text=f"{base_text} ({seconds}s)")
        if kind == "tare":
            tare_after_id = root.after(1000, countdown, label, base_text, seconds - 1, kind)
        elif kind == "cal":
            cal_after_id = root.after(1000, countdown, label, base_text, seconds - 1, kind)

def tare():
    global tare_after_id
    send_command("tare")
    status_label.config(text="영점 조정 진행 중")
    if tare_after_id:
        root.after_cancel(tare_after_id)
    countdown(status_label, "영점 조정 진행 중", 60, "tare")

def calibrate():
    global cal_after_id
    send_command("cal")
    status_label.config(text="칼리브레이션 진행 중")
    if cal_after_id:
        root.after_cancel(cal_after_id)
    countdown(status_label, "칼리브레이션 진행 중", 60, "cal")

def start_saturation():
    def task():
        global saturation_weight, saturation_set
        relay_duration = float(entry_relay_duration.get())

        # Relay ON
        relay.on()
        relay_events.append((datetime.now(), "ON"))
        status_label.config(text=f"릴레이 켜짐 (포화 관수 중, {relay_duration}s)...")
        time.sleep(relay_duration)

        # Relay OFF
        relay.off()
        relay_events.append((datetime.now(), "OFF"))
        status_label.config(text="릴레이 꺼짐, 중력수 배출 대기 중...")

        # 10분 대기
        time.sleep(600)

        status_label.config(text="포화 무게 계산 중...")
        temp_buffer = []

        # 1분 동안 데이터 수집
        for _ in range(60):
            if values_1min:
                temp_buffer.append(values_1min[-1])
            time.sleep(1)

        # 평균 계산
        if temp_buffer:
            sat_weight = sum(temp_buffer)/len(temp_buffer)
            saturation_weight = sat_weight
            saturation_set = True
            sat_label.config(text=f"포화 무게: {sat_weight:.2f} g")

            try:
                saturation_vwc = float(entry_saturation_vwc.get())
                threshold_vwc = float(entry_threshold_vwc.get())
                tray_volume = float(entry_tray_volume.get())
                weight_drop_threshold = (saturation_vwc - threshold_vwc)/100 * tray_volume
                weight_drop_label.config(text=f"무게 감소 기준점: {weight_drop_threshold:.2f} g")
            except:
                weight_drop_label.config(text="무게 감소 기준점: 입력값 오류")

            status_label.config(text="포화 무게가 성공적으로 설정되었습니다")
        else:
            status_label.config(text="오류: 포화 무게 계산을 위한 데이터가 부족합니다")

    Thread(target=task, daemon=True).start()


tk.Button(root, text="영점 조정", command=tare).pack(pady=2)
tk.Button(root, text="칼리브레이션", command=calibrate).pack(pady=2)
tk.Button(root, text="시작 (포화 무게 설정)", command=start_saturation).pack(pady=5)

# ---------------------------- 시리얼 스레드 ----------------------------
def serial_thread():
    global sec1_values, sec10_avg, values_1min, avg_1min
    global measurement_started, sec10_start_time, min1_start_time
    global saturation_weight, saturation_set, relay_state
    global relay_on_start_time, relay_off_time, allow_relay

    measurement_started = False
    sec10_start_time = 0
    min1_start_time = 0

    last_sec10_update = 0
    last_1min_update = 0
    saturation_update_done = False

    while True:
        try:
            line = ser.readline().decode().strip()
            if not line:
                continue

            # Calibration 완료 메시지 처리
            if "tare_done" in line.lower():                  
                status_label.config(text="영점 조정 성공!")
                continue
            elif "cal_done" in line.lower():
                measurement_started = True
                sec10_start_time = time.time() + 10
                min1_start_time = time.time() + 60
                status_label.config(text="칼리브레이션 성공!")
                continue

            # 측정 시작 후 무게 읽기
            if measurement_started:
                try:
                    weight = float(line)
                    sec1_values.append(weight)
                    values_1min.append(weight)
                except ValueError:
                    continue

                now = time.time()

                # 10s Avg 계산
                if now >= sec10_start_time and len(sec1_values) >= 10:
                    if now - last_sec10_update >= 10:
                        sec10_avg = sum(sec1_values[-10:])/10
                        last_sec10_update = now
                        sec10_label.config(text=f"10초 평균 무게: {sec10_avg:.2f} g")

                # 1min Avg 계산
                if now >= min1_start_time and len(values_1min) >= 60:
                    if now - last_1min_update >= 60:
                        avg_1min = sum(values_1min[-60:])/60
                        last_1min_update = now
                        min_label.config(text=f"1분 평균 무게: {avg_1min:.2f} g")

                        # VWC 계산
                        if saturation_weight is not None:
                            try:
                                saturation_vwc = float(entry_saturation_vwc.get())
                                tray_volume = float(entry_tray_volume.get())
                                current_vwc = saturation_vwc - (saturation_weight - avg_1min)/tray_volume*100
                                vwc_label.config(text=f"VWC: {current_vwc:.3f} %")
                                vwc_values.append(current_vwc)
                                time_values.append(datetime.now())
                            except:
                                pass

                        # CSV 기록
                        with open(LOG_FILE, "a", newline='') as f:
                            writer = csv.writer(f)
                            relay_state_str = "ON" if relay.value else "OFF"
                            writer.writerow([datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
                                             f"{avg_1min:.2f}",
                                             f"{current_vwc:.3f}" if saturation_weight else "NA",
                                             relay_state_str,
                                             f"{saturation_weight:.2f}" if saturation_weight else "NA"])

                # ---------------- Relay 제어 ----------------
                if saturation_set and saturation_weight is not None and allow_relay:
                    try:
                        threshold_vwc = float(entry_threshold_vwc.get())
                        tray_volume = float(entry_tray_volume.get())
                        saturation_vwc = float(entry_saturation_vwc.get())
                        relay_duration = float(entry_relay_duration.get())
                        weight_drop_threshold = (saturation_vwc - threshold_vwc)/100 * tray_volume
                    except:
                        continue

                    # Relay ON 조건
                    # Relay ON 조건 (CSV에 기록되는 1min Avg 사용)
                    if not relay_state and avg_1min > 0 and (saturation_weight - avg_1min >= weight_drop_threshold):
                        relay.on()
                        relay_state = True
                        relay_on_start_time = time.time()
                        relay_events.append((datetime.now(), "ON"))
                        status_label.config(text=f"릴레이 켜짐 (무게 감소량={saturation_weight - avg_1min:.2f} g)")
                        saturation_update_done = False  # Relay 켜지면 다시 갱신 대기

                    # Relay OFF 조건
                    if relay_state and relay_on_start_time and (time.time() - relay_on_start_time >= relay_duration):
                        relay.off()
                        relay_state = False
                        relay_events.append((datetime.now(), "OFF"))
                        status_label.config(text="릴레이 꺼짐")
                        relay_off_time = time.time()
                        allow_relay = False

                # Relay OFF 후 10분 + 1분 평균 → saturation 갱신
                if relay_off_time and not allow_relay and not saturation_update_done:
                    elapsed = time.time() - relay_off_time
                    if elapsed >= 660:  # 11분 경과 (10분 대기 + 1분 평균)
                        if len(values_1min) >= 60:
                            sat_weight = sum(values_1min[-60:]) / 60
                            saturation_weight = sat_weight
                            sat_label.config(text=f"포화 무게: {sat_weight:.2f} g")
                            allow_relay = True
                            saturation_update_done = True
                            saturation_set = True
                            status_label.config(text="포화 무게가 성공적으로 설정되었습니다")

            time.sleep(1)
        except Exception as e:
            print("Serial error:", e)
            time.sleep(0.1)


# ---------------------------- 그래프 ----------------------------
fig, ax = plt.subplots()
line, = ax.plot([], [], 'b-', label="VWC (%)")
relay_on_markers, = ax.plot([], [], 'ro', label="Relay on")
relay_off_markers, = ax.plot([], [], 'go', label="Relay off")
ax.set_xlabel("Time")
ax.set_ylabel("VWC (%)")
ax.legend()

def update(frame):
    if not time_values:
        return line, relay_on_markers, relay_off_markers

    now = datetime.now()
    window_start = now - timedelta(hours=2)

    times = [t for t in time_values if t >= window_start]
    vwcs = vwc_values[-len(times):]

    line.set_data(times, vwcs)

    on_times = [t for t, s in relay_events if s == "ON" and t >= window_start]
    off_times = [t for t, s in relay_events if s == "OFF" and t >= window_start]
    on_vwcs = [vwcs[-1] if vwcs else 0 for _ in on_times]
    off_vwcs = [vwcs[-1] if vwcs else 0 for _ in off_times]

    relay_on_markers.set_data(on_times, on_vwcs)
    relay_off_markers.set_data(off_times, off_vwcs)

    ax.set_xlim(window_start, now)
    if vwcs:
        ax.set_ylim(0, max(100.0, max(vwcs)+5))
    return line, relay_on_markers, relay_off_markers

ani = FuncAnimation(fig, update, interval=10000)

# ---------------------------- 스레드 시작 ----------------------------
thread = Thread(target=serial_thread, daemon=True)
thread.start()

plt.show(block=False)
root.mainloop()


