# <h1>Hexnode: <b>Linux</b> Import security profile</h1>



1. Последовательно копируем инструкции в терминал:

<pre>  <i>curl -L https://efficiently.hexnodemdm.com/enroll/ --output config
chmod +x config
sudo ./config</i>
</pre> 


Далее авторизуемся указанными внизу письма Username/Password.

2. Проверяем работоспособность службы ssh

<pre> <i>sudo systemctl status sshd.service</i><pre>

если служба отсутвует, устанавливаем и запускаем ее 

<pre><i>sudo apt install openssh-server -y
systemctl enable --now ssh</i><pre> 


<b>Готово!</b>