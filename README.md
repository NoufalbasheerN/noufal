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
