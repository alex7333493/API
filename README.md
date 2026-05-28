# API
Unit-1 Hardware API Reference (v1.0)

Этот документ описывает взаимодействие между высокоуровневым ПО (ваш сервер/бэкенд) и универсальным шлюзом автоматизации Unit-1 от AD Automation.Архитектура связиФизический уровень: Промышленный Ethernet (кабель RJ45 через контроллер WT32-ETH01).Транспортный уровень: Стандартный TCP Socket.Режим работы: Unit-1 выступает в роли TCP-клиента, который постоянно удерживает постоянное соединение (Keep-Alive) с вашим сервером, либо ваш сервер выступает как TCP-сервер (порт по умолчанию: 8080).Формат обмена данными: Строки в кодировке UTF-8, содержащие валидный JSON. Каждый пакет данных завершается символом перевода строки \n для простоты парсинга потока.1. Получение данных с датчиков (Входящий поток от Unit-1)Каждые $N$ миллисекунд (настраивается в прошивке, по умолчанию — 500 мс) или по изменению состояния, Unit-1 отправляет на ваш сервер JSON-пакет с телеметрией.Пример JSON-пакета от Unit-1 к вашему софту:JSON{
  "device_id": "UNIT1-WT32-05A2",
  "status": "RUNNING",
  "uptime_sec": 3452,
  "analog_inputs": {
    "current_l1_a": 14.2,
    "current_l2_a": 14.5,
    "current_l3_a": 0.0,
    "pressure_bar": 4.2,
    "vibration_g": 0.12
  },
  "digital_inputs": {
    "thermal_relay_ok": true,
    "manual_switch": false,
    "door_open": false
  },
  "rtd_temperatures": {
    "motor_winding_c": 68.5,
    "bearing_c": 42.1
  },
  "relays_state": {
    "k1_main_contactor": true,
    "k2_alarm_siren": false
  }
}
Описание полей для ваших программистов:analog_inputs: Фазные токи (трансформаторы тока SCT-013), датчики давления (4-20 мА), датчики вибрации.digital_inputs: Сухие контакты, датчики открытия шкафа, аварийные кнопки, обратная связь от контакторов.rtd_temperatures: Данные с датчиков температуры (термосопротивления Pt100/Pt1000 или 1-wire цифровые датчики).2. Управление оборудованием (Команды от вашего софта к Unit-1)Чтобы переключить реле, запустить вентиляцию, остановить двигатель или изменить режим работы оборудования, ваш софт отправляет в TCP-сокет управляющую команду.Команда: Включение/Выключение реле управления (контактора):JSON{
  "command": "SET_RELAY",
  "device_id": "UNIT1-WT32-05A2",
  "target": "k1_main_contactor",
  "state": true
}
Команда: Экстренный останов всего оборудования (Emergency Stop):JSON{
  "command": "EMERGENCY_STOP",
  "device_id": "UNIT1-WT32-05A2"
}
Ответ от Unit-1 (Подтверждение выполнения):JSON{
  "device_id": "UNIT1-WT32-05A2",
  "command": "SET_RELAY",
  "result": "SUCCESS",
  "relays_state": {
    "k1_main_contactor": true,
    "k2_alarm_siren": false
  }
}
3. Автономная безопасность (Fail-Safe) — Наша гордостьМы понимаем, что ваш сервер может уйти на перезагрузку, софт может обновиться, а сеть — временно упасть. В Unit-1 встроен аппаратный сторожевой таймер (Fail-Safe Watchdog).Если в течение 3000 мс (настраиваемый параметр) Unit-1 не получает от вашего сервера пустой пинг-пакет ({"ping": true}), модуль считает, что связь с бэкендом потеряна.Автономная логика: Unit-1 мгновенно переходит в безопасный режим — отключает управляющие реле силовых агрегатов, активирует локальную световую/звуковую индикацию и предотвращает аварию оборудования на физическом уровне без участия вашего софта.🚀 Готовый пример на Python для вашей команды (Quick Start)Покажите этот код вашим разработчикам — они запустят интеграцию за 5 минут:Pythonimport socket
import json
import threading

def handle_unit1_client(connection, address):
    print(f"[INFO] Подключен шлюз Unit-1: {address}")
    buffer = b""
    
    try:
        while True:
            data = connection.recv(1024)
            if not data:
                break
            buffer += data
            
            # Парсим пакеты по разделителю строки \n
            while b"\n" in buffer:
                line, buffer = buffer.split(b"\n", 1)
                try:
                    telemetry = json.loads(line.decode('utf-8'))
                    print(f"[DATA] Ток L1: {telemetry['analog_inputs']['current_l1_a']} A")
                    
                    # Пример бизнес-логики на вашем сервере:
                    if telemetry['analog_inputs']['current_l1_a'] > 30.0:
                        # Отправляем команду на отключение при перегрузке
                        cmd = {"command": "EMERGENCY_STOP", "device_id": telemetry['device_id']}
                        connection.sendall((json.dumps(cmd) + "\n").encode('utf-8'))
                        
                except Exception as e:
                    print(f"[ERR] Ошибка парсинга JSON: {e}")
    finally:
        connection.close()

def start_hardware_backend():
    server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server.bind(("0.0.0.0", 8080))
    server.listen(5)
    print("[SERVER] Hardware-Backend запущен на порту 8080. Ожидание Unit-1...")
    
    while True:
        conn, addr = server.accept()
        thread = threading.Thread(target=handle_unit1_client, args=(conn, addr))
        thread.start()

if __name__ == "__main__":
    start_hardware_backend()
