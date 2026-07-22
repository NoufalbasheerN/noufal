# noufal
scp "C:\Users\Admin\Downloads\fwlib-master\fwlib-master\libfwlib32-linux-x64.so.1.0.5" lgb_iot_team@10.100.6.100:/home/lgb_iot_team/

sudo nano /etc/systemd/system/cnc_monitor.service
[Unit]
Description=CNC Monitor Service
After=network.target

[Service]
User=YOUR_USERNAME
WorkingDirectory=/home/YOUR_USERNAME
ExecStart=/usr/bin/python3 /home/YOUR_USERNAME/cnc_monitor.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
sudo systemctl daemon-reload
sudo systemctl enable cnc_monitor.service
sudo systemctl start cnc_monitor.service
sudo systemctl status cnc_monitor.service



import ctypes
import os
import time
from datetime import datetime
from pymongo import MongoClient

# =====================================================
# MONGODB
# =====================================================
client = MongoClient(
    "mongodb://OTAdmin:vGF44j%23%265K%266@10.100.6.100:27017/admin",
    serverSelectionTimeoutMS=3000
)

db = client["cnc_monitoring"]
collection = db["production"]

# =====================================================
# FOCAS LIBRARY
# =====================================================
LIB_PATH = os.path.expanduser("~/libfwlib32-linux-x64.so.1.0.5")
focas = ctypes.CDLL(LIB_PATH)

# =====================================================
# STARTUP
# =====================================================
ret = focas.cnc_startupprocess(3, b"focas.log")
if ret != 0:
    print("FOCAS Startup Error:", ret)
    raise SystemExit(1)

# =====================================================
# MACHINES
# =====================================================
MACHINES = [
    {"name": "GR-TU-68", "ip": b"10.100.6.115"},
    {"name": "GR-TU-67", "ip": b"10.100.6.116"},
    {"name": "GR-TU-61", "ip": b"10.100.6.117"},
]

# =====================================================
# STRUCTURES
# =====================================================
class ODBM(ctypes.Structure):
    _pack_ = 1
    _fields_ = [
        ("datano", ctypes.c_int16),
        ("dummy", ctypes.c_int16),
        ("mcr_val", ctypes.c_int32),
        ("dec_val", ctypes.c_int16),
    ]


class ODBST(ctypes.Structure):
    _fields_ = [
        ("dummy", ctypes.c_short),
        ("aut", ctypes.c_short),
        ("run", ctypes.c_short),
        ("motion", ctypes.c_short),
        ("mstb", ctypes.c_short),
        ("emergency", ctypes.c_short),
        ("alarm", ctypes.c_short),
        ("edit", ctypes.c_short),
    ]


class IODBTIME(ctypes.Structure):
    _pack_ = 1
    _fields_ = [
        ("minute", ctypes.c_int32),
        ("msec", ctypes.c_int32),
    ]


def format_timer(minutes, msec):
    total_seconds = minutes * 60 + (msec // 1000)
    hh = total_seconds // 3600
    mm = (total_seconds % 3600) // 60
    ss = total_seconds % 60
    return f"{hh:02d}:{mm:02d}:{ss:02d}"


# =====================================================
# CONNECT CNC
# =====================================================
handles = {}

for machine in MACHINES:
    handle = ctypes.c_ushort()

    ret = focas.cnc_allclibhndl3(
        machine["ip"],
        8193,
        10,
        ctypes.byref(handle)
    )

    if ret == 0:
        handles[machine["name"]] = handle
        print(f"{machine['name']} Connected")
    else:
        print(f"{machine['name']} Failed : {ret}")

# =====================================================
# CYCLE TRACKING
# =====================================================
prev_cycle_seconds = {}
cycle_active = {}

for m in MACHINES:
    prev_cycle_seconds[m["name"]] = 0
    cycle_active[m["name"]] = False

# =====================================================
# MAIN LOOP
# =====================================================
while True:

    for machine_name, handle in handles.items():

        try:

            # PART COUNT
            part = ODBM()
            ret = focas.cnc_rdmacro(
                handle.value,
                3901,
                10,
                ctypes.byref(part)
            )

            if ret == 0:
                part_count = int(part.mcr_val / (10 ** abs(part.dec_val)))
            else:
                part_count = -1

            # TARGET COUNT
            target = ODBM()
            ret = focas.cnc_rdmacro(
                handle.value,
                3902,
                10,
                ctypes.byref(target)
            )

            if ret == 0:
                target_count = int(target.mcr_val / (10 ** abs(target.dec_val)))
            else:
                target_count = -1

            # STATUS
            status = ODBST()
            ret = focas.cnc_statinfo(
                handle.value,
                ctypes.byref(status)
            )

            alarm = status.alarm if ret == 0 else -1

            # TIMERS
            timer0 = IODBTIME()
            focas.cnc_rdtimer(handle.value, 0, ctypes.byref(timer0))

            timer1 = IODBTIME()
            focas.cnc_rdtimer(handle.value, 1, ctypes.byref(timer1))

            timer2 = IODBTIME()
            focas.cnc_rdtimer(handle.value, 2, ctypes.byref(timer2))

            timer3 = IODBTIME()
            focas.cnc_rdtimer(handle.value, 3, ctypes.byref(timer3))

            power_on = format_timer(timer0.minute, timer0.msec)
            run_time = format_timer(timer1.minute, timer1.msec)
            cutting_time = format_timer(timer2.minute, timer2.msec)

            cycle_seconds = (
                timer3.minute * 60 +
                timer3.msec // 1000
            )

            # Cycle started
            if cycle_seconds > 0:
                cycle_active[machine_name] = True

            # Cycle completed when timer resets to 0
            if (
                cycle_active[machine_name]
                and prev_cycle_seconds[machine_name] > 0
                and cycle_seconds == 0
            ):

                completed_cycle = prev_cycle_seconds[machine_name]

                hh = completed_cycle // 3600
                mm = (completed_cycle % 3600) // 60
                ss = completed_cycle % 60

                actual_cycle_time = f"{hh:02d}:{mm:02d}:{ss:02d}"

                power_seconds = (
                    timer0.minute * 60 +
                    timer0.msec // 1000
                )

                run_seconds = (
                    timer1.minute * 60 +
                    timer1.msec // 1000
                )

                down_seconds = max(
                    power_seconds - run_seconds,
                    0
                )

                dh = down_seconds // 3600
                dm = (down_seconds % 3600) // 60
                ds = down_seconds % 60

                machine_down_time = (
                    f"{dh:02d}:{dm:02d}:{ds:02d}"
                )

                document = {
                    "timestamp": datetime.now(),
                    "machine_name": machine_name,
                    "machine_cycle_time": actual_cycle_time,
                    "machine_down_time": machine_down_time,
                    "overall_run_time": run_time,
                    "total_part_count": part_count,
                    "ok_part_count": part_count,
                    "not_ok_part_count": 0,
                    "target_count": target_count,
                    "alarm": alarm,
                    "power_on": power_on,
                    "cutting_time": cutting_time
                }

                collection.insert_one(document)

                print("=" * 60)
                print(f"{machine_name} CYCLE COMPLETED")
                print(f"Cycle Time   : {actual_cycle_time}")
                print(f"Part Count   : {part_count}")
                print(f"Target Count : {target_count}")
                print("MongoDB Saved")
                print("=" * 60)

                cycle_active[machine_name] = False

            prev_cycle_seconds[machine_name] = cycle_seconds

        except Exception as e:
            print(f"{machine_name} Error : {e}")

    time.sleep(5)
