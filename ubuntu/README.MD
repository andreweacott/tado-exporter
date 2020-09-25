sudo cp ./tado-exporter.service /etc/systemd/system/tado-exporter.service
sudo mkdir -p /etc/tado-exporter
sudo cp ./tado-exporter.env /etc/tado-exporter/tado-exporter.env
sudo systemctl daemon-reload
