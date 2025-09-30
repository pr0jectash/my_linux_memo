# PipeWire + Ardour + BrowserSink + Firefox ルーティング 手順まとめ

## 1. JACKをPipeWireで置き換え
sudo apt install pipewire pipewire-pulse pipewire-jack wireplumber qpwgraph
# PipeWire本体・Pulse互換・JACK互換・セッションマネージャ・接続GUIを導入

## 2. サービス有効化
systemctl --user enable --now pipewire pipewire-pulse wireplumber
# ログイン時に自動起動するよう設定

## 3. 仮想デバイス BrowserSink を常駐化
mkdir -p ~/.config/systemd/user
nano ~/.config/systemd/user/browser-sink.service

# 以下を記述：
# [Unit]
# Description=Create BrowserSink
# After=pipewire-pulse.service
#
# [Service]
# ExecStart=/usr/bin/pactl load-module module-null-sink sink_name=BrowserSink sink_properties=device.description=BrowserSink
# RemainAfterExit=true
#
# [Install]
# WantedBy=default.target

systemctl --user enable --now browser-sink.service
# ログイン時に自動で BrowserSink が作成される

## 4. Firefox の出力先を BrowserSink と UR22C に設定
sudo apt install pavucontrol
pavucontrol
# 「再生」タブで Firefox の出力先を BrowserSink に変更
# さらにFirefoxの出力先を UR22C にも設定
# Pulse/PipeWire は一度選んだ出力先を記憶するので、次回以降も自動で接続される

## 5. Ardour 側の入力設定
# Ardour で録音用トラックを作成し、入力を BrowserSink.monitor_L / BrowserSink.monitor_R に設定
# セッションを保存すれば次回も維持される

# --- 最終的なルーティング ---
# Firefox → UR22C（モニター出力）
# Firefox → BrowserSink → Ardour（録音）




# PipeWire + Ardour + BrowserSink 構築で遭遇したトラブルと解決策

## 1. JACK と PipeWire の共存問題
- 現象:
  Ardour を JACK モードで起動すると、jackd2 の libjack が優先され PipeWire に流れない。
- 解決策:
  /usr/lib/x86_64-linux-gnu/pipewire-0.3/jack を ld.so.conf.d に追加し ldconfig。
ls -l /usr/lib/x86_64-linux-gnu/pipewire-0.3/jack 
ライブラリの確認
ldconfig -p | grep libjack
優先されてるものを確認

echo "/usr/lib/x86_64-linux-gnu/pipewire-0.3/jack" | sudo tee /etc/ld.so.conf.d/pipewire-jack-x86_64-linux-gnu.conf
pipewireのjackライブラリへのパスを追加。

/lib/x86_64-linux-gnu/libjack.so.0これが最初にpipwireが読んでたのが原因。
ファイルを追加して、文字順で追加したものが優先されるようになった。

sudo ldconfig キャッシュを更新
ldconfig -p | grep libjack
 更新されてるか確認
  
  ※ libjack-jackd2-0 を削除すると Ardour/ffmpeg などが巻き添えで消えるため再インストールで復旧。

## 2. Ardour 入力が毎回リセットされる
- 現象:
  Firefox は音を出している間だけノードが存在するため、止めると Ardour の入力が system capture に戻る。
- 解決策:
  pactl module-null-sink で BrowserSink を常駐化。
  Ardour の入力を BrowserSink.monitor に固定し、セッション保存で安定化。

## 3. pipewire-pulse サービスの起動失敗
- 現象:
  browser-sink.conf に誤ったモジュール名を書いた結果、
  「could not load mandatory module "libpipewire-module-null-audio-sink"」エラーで pipewire-pulse が起動不能。
  pactl が「接続拒否」となり音が出なくなる。
- 解決策:
  設定ファイルを退避し、pipewire/pipewire-pulse/wireplumber を再起動して復旧。

## 4. Firefox が落ちる
- 現象:
  WirePlumber の Lua ルールで Firefox の出力を BrowserSink に強制しようとしたが、プロパティ指定が合わず Firefox がクラッシュ。
- 解決策:
  ルールを削除して再起動。
  最終的には pavucontrol で Firefox の出力を BrowserSink に一度切り替えると記憶され、安定運用可能。
