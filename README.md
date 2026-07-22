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
exit()
# =====================================================
# MACHINES
# =====================================================
MACHINES = [
{"name": "CNC1", "ip": b"10.100.5.21"},
{"name": "CNC2", "ip": b"10.100.5.22"},
{"name": "CNC3", "ip": b"10.100.5.23"},
{"name": "CNC4", "ip": b"10.100.5.24"}
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
("dec_val", ctypes.c_int16) ]
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
# =====================================================
# CONNECT ALL MACHINES
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
print(f"{machine['name']} Failed : {ret}") # =====================================================
# HELPER
# =====================================================
def format_timer(minutes, msec):
total_seconds = (minutes * 60) + (msec // 1000)
hh = total_seconds // 3600
mm = (total_seconds % 3600) // 60
ss = total_seconds % 60
return f"{hh:02d}:{mm:02d}:{ss:02d}"
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
# ==================================
# PART COUNT
# ==================================
part = ODBM()
ret = focas.cnc_rdmacro(
handle.value,
3901,
10,
ctypes.byref(part)
) if ret == 0:
part_count = int(
part.mcr_val /
(10 ** abs(part.dec_val))
)
else:
part_count = -1
# ==================================
# TARGET COUNT
# ==================================
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
# ==================================
# STATUS
# ==================================
status = ODBST()
ret = focas.cnc_statinfo(
handle.value,
ctypes.byref(status)
)
if ret == 0:
if status.motion == 3:
machine_state = "RUNNING" else:
machine_state = "STOPPED"
alarm = status.alarm
else:
machine_state = "UNKNOWN"
alarm = -1
# ==================================
# TIMER 0
# ==================================
timer0 = IODBTIME()
ret = focas.cnc_rdtimer(
handle.value,
0,
ctypes.byref(timer0)
)
power_on = format_timer(
timer0.minute,
timer0.msec
) if ret == 0 else "NA"
# ==================================
# TIMER 1
# ==================================
timer1 = IODBTIME()
ret = focas.cnc_rdtimer(
handle.value,
1,
ctypes.byref(timer1)
)
run_time = format_timer(
timer1.minute,
timer1.msec
) if ret == 0 else "NA"
# ================================== # TIMER 2
# ==================================
timer2 = IODBTIME()
ret = focas.cnc_rdtimer(
handle.value,
2,
ctypes.byref(timer2)
)
cutting_time = format_timer(
timer2.minute,
timer2.msec
) if ret == 0 else "NA"
# ==================================
# TIMER 3
# ==================================
timer3 = IODBTIME()
ret = focas.cnc_rdtimer(
handle.value,
3,
ctypes.byref(timer3)
)
cycle_time = format_timer(
timer3.minute,
timer3.msec
) if ret == 0 else "NA"
# ==================================
# DISPLAY
# ==================================
print(f"\n[{machine_name}]")
print("Part Count :", part_count)
print("Target Count :", target_count)
print("State :", machine_state)
print("Alarm :", alarm)
print("Power On :", power_on)
print("Run Time :", run_time) print("Cutting Time :", cutting_time)
print("Cycle Time :", cycle_time)
except Exception as e:
print(f"{machine_name} Error :", e)
print("\nRefreshing every 5 seconds...")
time.sleep(5)
# =====================================================
# CLOSE
# =====================================================
for handle in handles.values():
focas.cnc_freelibhndl(handle.value)
focas.cnc_exitprocess()
