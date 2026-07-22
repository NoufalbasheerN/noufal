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
MONGO_IP = "10.100.6.100"
MONGO_PORT = 27017
client = MongoClient(
f"mongodb://OTAdmin:vGF44j%23%265K%266@10.100.6.100:27017/admin",
serverSelectionTimeoutMS=3000
)
db = client["cnc_monitoring"]
collection = db["production"]
# =====================================================
# FOCAS LIBRARY
# =====================================================
LIB_PATH = os.path.expanduser(
"~/libfwlib32-linux-x64.so.1.0.5"
)
focas = ctypes.CDLL(LIB_PATH)
# =====================================================
# STARTUP
# =====================================================
ret = focas.cnc_startupprocess(
3,
b"focas.log"
)
if ret != 0:
print("FOCAS Startup Error:", ret)
exit() # =====================================================
# MACHINES
# =====================================================
MACHINES = [
{"name": "GR-TU-68", "ip": b"10.100.6.115"},
{"name": "GR-TU-67", "ip": b"10.100.6.116"},
{"name": "GR-TU-61", "ip": b"10.100.6.117"}
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
("dec_val", ctypes.c_int16)
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
("edit", ctypes.c_short)
]
class IODBTIME(ctypes.Structure):
_pack_ = 1
_fields_ = [
("minute", ctypes.c_int32),
("msec", ctypes.c_int32)
]
# ===================================================== # HELPERS
# =====================================================
def format_timer(minutes, msec):
total_seconds = (
minutes * 60 +
(msec // 1000)
)
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
previous_state = {} cycle_start_time = {}
machine_on_count = {}
machine_off_count = {}
for m in MACHINES:
previous_state[m["name"]] = "UNKNOWN"
cycle_start_time[m["name"]] = None
machine_on_count[m["name"]] = 0
machine_off_count[m["name"]] = 0
# =====================================================
# MAIN LOOP
# =====================================================
while True:
os.system("clear")
print("=" * 100)
print("FANUC PRODUCTION MONITOR")
print(datetime.now())
print("=" * 100)
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
part_count = int(
part.mcr_val /
(10 ** abs(part.dec_val))
) else:
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
target_count = int(
target.mcr_val /
(10 ** abs(target.dec_val))
)
else:
target_count = -1
# STATUS
status = ODBST()
ret = focas.cnc_statinfo(
handle.value,
ctypes.byref(status)
)
if ret == 0:
if status.motion == 3:
machine_state = "RUNNING"
else:
machine_state = "STOPPED"
alarm = status.alarm
else:
machine_state = "UNKNOWN"
alarm = -1 # POWER ON
timer0 = IODBTIME()
ret = focas.cnc_rdtimer(
handle.value,
0,
ctypes.byref(timer0)
)
power_on = (
format_timer(
timer0.minute,
timer0.msec
)
if ret == 0 else "NA"
)
# RUN TIME
timer1 = IODBTIME()
ret = focas.cnc_rdtimer(
handle.value,
1,
ctypes.byref(timer1)
)
run_time = (
format_timer(
timer1.minute,
timer1.msec
)
if ret == 0 else "NA"
)
# CUTTING TIME
timer2 = IODBTIME()
ret = focas.cnc_rdtimer(
handle.value,
2, ctypes.byref(timer2)
)
cutting_time = (
format_timer(
timer2.minute,
timer2.msec
)
if ret == 0 else "NA"
)
# CYCLE TIMER
timer3 = IODBTIME()
ret = focas.cnc_rdtimer(
handle.value,
3,
ctypes.byref(timer3)
)
cycle_time = (
format_timer(
timer3.minute,
timer3.msec
)
if ret == 0 else "NA"
)
current_time = datetime.now()
# ==================================
# CYCLE START
# ==================================
if (
previous_state[machine_name] != "RUNNING"
and
machine_state == "RUNNING"
):
cycle_start_time[machine_name] = current_time
machine_on_count[machine_name] += 1 # ==================================
# CYCLE STOP
# ==================================
elif (
previous_state[machine_name] == "RUNNING"
and
machine_state == "STOPPED"
):
machine_off_count[machine_name] += 1
if cycle_start_time[machine_name]:
cycle_stop_time = current_time
cycle_seconds = int(
(
cycle_stop_time -
cycle_start_time[machine_name]
).total_seconds()
)
hh = cycle_seconds // 3600
mm = (cycle_seconds % 3600) // 60
ss = cycle_seconds % 60
actual_cycle_time = (
f"{hh:02d}:"
f"{mm:02d}:"
f"{ss:02d}"
)
power_seconds = (
timer0.minute * 60 +
timer0.msec // 1000
)
run_seconds = (
timer1.minute * 60 +
timer1.msec // 1000
)
down_seconds = max( power_seconds -
run_seconds,
0
)
dh = down_seconds // 3600
dm = (down_seconds % 3600) // 60
ds = down_seconds % 60
machine_down_time = (
f"{dh:02d}:"
f"{dm:02d}:"
f"{ds:02d}"
)
document = {
"timestamp": current_time,
"machine_name":
machine_name,
"cycle_start":
cycle_start_time[
machine_name
],
"cycle_stop":
cycle_stop_time,
"machine_cycle_time":
actual_cycle_time,
"machine_down_time":
machine_down_time,
"overall_run_time":
run_time,
"total_part_count":
part_count,
"ok_part_count":
part_count, "not_ok_part_count":
0,
"machine_on_count":
machine_on_count[
machine_name
],
"machine_off_count":
machine_off_count[
machine_name
],
"target_count":
target_count,
"alarm":
alarm
}
collection.insert_one(
document
)
print(
f"{machine_name} "
f"Cycle Saved"
)
previous_state[machine_name] = machine_state
print(f"\n[{machine_name}]")
print("Part Count :", part_count)
print("Target Count :", target_count)
print("State :", machine_state)
print("Alarm :", alarm)
print("Power On :", power_on)
print("Run Time :", run_time)
print("Cutting Time :", cutting_time)
print("Cycle Time :", cycle_time)
except Exception as e: print(
f"{machine_name} Error : {e}"
)
time.sleep(5)
            
