# esp32-cam-mb
Configuration for ESP32-CAM-MB for the ESPHome + Frigate + Home Assistant

# Primeiro passo - comunicação do ESP32-CAM-MB com o computador:
- Verifique o chip de comunicação Serial/USB existente na placa. Em seguida, instalar o driver para o sistema operacional desejado (Win / Mac).
* CP2102:
  -> Windows e Mac: https://www.silabs.com/products/development-tools/software/usb-to-uart-bridge-vcp-drivers
* CH342, CH343, CH9102:
  -> Windows https://www.wch.cn/downloads/CH343SER_ZIP.html
  -> Mac (https://www.wch.cn/downloads/CH34XSER_MAC_ZIP.html)
* CH340, CH341 drivers:
  -> Windows - https://www.wch.cn/downloads/CH341SER_ZIP.html
  -> Mac - https://www.wch.cn/downloads/CH341SER_MAC_ZIP.html
- Conectar o ESP32-CAM-MB no computador via USB.
*Dica:* Confirmar se cabo utilizado é do tipo *Dados + Energia*, pois existem cabos com trilha para apenas Energia.
- Abrir o site do ESP Home (https://web.esphome.io/) por algum navegador que suporte WebSerial.
- Selecionar a porta USB Serial (COM x).
- Se tudo estiver funcionando corretamente, vai aparecer a mensagem "connected" ao lado do nom "ESP Device".

# Segundo passo - preparar o dispostivo para uso
- Clicar em "prepare for first use" e avançar

# Terceiro passo - configurar os dados de WI-FI
- Clicar no "..." ao lado de "Logs" e, em seguida, "Configure WI-FI". Inserir os dados de SSID e Password e realizar a conexão.
*Observação:* os dados de SSID e Password devem estar registrados no arquivo "secrets" dentro do ESP Home (Sidebar -> ESP Home (dev) -> Upperbar - Secrets).
- substituir "xxxxx" pelos dados corretos e clicar em salvar.

		wifi_ssid: "xxxxx"
		wifi_password: "xxxxx"

# Quarto passo - obter as chaves de encriptação da "api" e a chave do "ota"
- Abrir o ESP Home, clicar em "Novo dispositivo", clicar em "continue", adicionar o nome "ESP32-CAM-MB", selelecionar "ESP32", copiar a chave de encriptação e clicar em install - plug into this computer.
- Clicar em "Download Project"
- Voltar na aba do ESP Home *https://web.esphome.io/* e clicar em "install".
- Neste ponto, o arquivo de configuração "ESP32-CAM-MB" estará com as chaves (API e OTA) já preenchidas, bem como o firmware correspondente estará instalado.

# Quinta passo - ativar a câmera e o LED
- Ir na Sidebar -> ESP Home (dev) e clicar em "edit" dentro de "ESP32-CAM-MB".
- Adicionar as seguintes linhas e fazer os ajuste de pinagem e qualidade de imagem:

		esp32_camera:
		  name: "My Cam"
		  external_clock:
		    pin: GPIO0
		    frequency: 10MHz
		  i2c_pins:
		    sda: GPIO26
		    scl: GPIO27
		  data_pins: [GPIO5, GPIO18, GPIO19, GPIO21, GPIO36, GPIO39, GPIO34, GPIO35]
		  vsync_pin: GPIO25
		  href_pin: GPIO23
		  pixel_clock_pin: GPIO22
		  power_down_pin: GPIO32
		  wb_mode: AUTO
		  max_framerate: 12 fps   
		  idle_framerate: 1 fps 
		  resolution: 640x480 #320x240
		  jpeg_quality: 30
		
		switch:
		  - platform: gpio
		    name: "My Cam flash"
		    pin: 4

		  sensor:
		  - platform: wifi_signal
		    name: "WiFi ESP32-CAM"
		    update_interval: 60s

- Clicar em salvar e, em seguida, clicar em install - plug into this computer.
- Clicar em "Download Project"
- Voltar na aba do ESP Home *https://web.esphome.io/* e clicar em "install".
- Neste ponto, o firmware já permitirá que o Home Assistant tenha acesso à câmera e ao LED, bem como foi criada a entidade "WiFi ESP32-CAM" para monitorar a potência do sinal recebido na câmera. Este ponto é interessante ser observado, pois a qualidade/potência do sinal de wi-fi recebido influencia diretamente na quantidade de "FPS" recebido pelo Frigate e Home Assistant.
- Para verificar se a câmera está funcionando, volte ao ESP Home (dev) e clique em "logs" na caixa do ESP32-CAM-MB. O sistema vai gerar o seguinte log refente aos frames gerados.

		[13:21:00][D][esp32_camera:196]: Got Image: len=11666
		[13:21:00][D][esp32_camera:196]: Got Image: len=11654
		[13:21:00][D][esp32_camera:196]: Got Image: len=11697

*Bonus:* ajustar a qualidade de imagem desejada "max_framerate", "idle_framerate", "resolution" e "jpeg_quality". Fonte - https://theorangeone.net/posts/esp-home-camera/

![image](https://github.com/mariogppcom/esp32-cam-mb/assets/38541749/eb2156e1-aded-4979-8f02-e6adc081d922)

- Neste ponto, já é possível fazer o acionamento do LED, bem como verificar a imagem da câmera pelo Home Assistant.

# Sexto passo - permitir que o Frigate tenha acesso à câmera

- Ir na Sidebar -> ESP Home (dev) e clicar em "edit" dentro de "ESP32-CAM-MB" e adicionar as seguintes linhas:

		esp32_camera_web_server:
		  - port: 8080
		    mode: stream
		  - port: 8081
		    mode: snapshot

- Ir no Frigate, sidebar "config" e adicionar as linhas abaixo e ajustar: o IP da câmera nas linhas 4 e 11, ajustar "width: xxx", "height: xxx" e "fps: 2" conforme ajustado no quinto passo.

		go2rtc:
		  streams:
		      ESP32-CAM-MB:
		      - ffmpeg:http://IP DA CÂMERA:8080#video=h264
		
		cameras:
			ESP32-CAM-MB:  
				ffmpeg:  
					input_args: ""          
					inputs:  
						- path: - ffmpeg:http://IP DA CÂMERA:8080#video=h264
							roles:  
								- detect  
								- record
					output_args:  
					 record: -f segment -pix_fmt yuv420p -segment_time 10 -segment_format mp4 -reset_timestamps 1 -strftime 1 -c:v libx264 -preset ultrafast -an  
				rtmp:  
					enabled: False  
				snapshots:  
					enabled: true  
					bounding_box: true  
				record:  
					enabled: True  
					retain:  
						days: 0 # To not retain any recording if there is no detection of any events  
					events:  
						retain:  
							default: 3 # To retain recording for 3 days of only the events that happened  
							mode: active_objects 
				detect:
					enabled: True  
					width: 800  
					height: 600  
					fps: 2      #Adjust the fps based on what suits your hardware.  
				objects:
					track:  
						- person
  - Salve a aplique as configurações.

# Sétimo passo - adicionar a câmera no Home Assistant
- Instale o add-on "Frigate Proxy" (Home Assistant -> Configurações -> Add-ons -> Loja de Add-ons)
- Após instalar, clique em "Frigate Proxy", ajustes e verifique o ip e porta do servidor http://xxx.xxx.xxx.xxx:yyyy
- Instale a integração "Frigate" (Home Assistant -> Configurações -> Dispositivos & Serviços -> Integrações -> Adicionar integração)


# Bugs
- Algumas vezes "trava" o esp32-cam-mb. A única forma de "recuperar" é instalando o firmware novamente. Então organize (versione) os firmwares desenvolvidos.
- Em algum momentos o esp32-cam-mb apresenta alguns comportamentos estranhos, especialmente quando utilizado "carregador de celular". Parte destes comportamentos cessaram utiliziando um cabo USB full (dados + energia), mesmo que o cabo seja utilizado apenas para energizar a placa.
- Foi possível notar uma perda sensível de "FPS" gerados pela câmera de acordo a qualidade do sinal "WI-FI". Recomendo a compra do kit ESP32-CAM-MB (câmera + placa principal) com o kit antena para maximizar a qualidade do sinal e, por consequência, a permitir maiores distâncias entre a câmera e o roteador mais próximo.
