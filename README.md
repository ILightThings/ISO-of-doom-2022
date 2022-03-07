# ISO-of-doom-2022
A small read on how to build the Offsec ISO of doom, but actually make it work.


## Preamble
So work had asked me to make something after dealing with constant client VPN issues, to help us gain and maintain access to client networks with little interaction and low maintence. This is when I stumbled upon the annoying nusience that is Offensives Securitys version of Live Build for Kali.

A long time ago, Offensive Security put out an article named [The ISO of DOOM](https://www.offensive-security.com/kali-linux/kali-linux-iso-of-doom/). In their words,
> The idea we had was to build an “unattended self-deploying” instance of Kali Linux that would install itself on a target machine along with a customized configuration requiring no user input whatsoever. On reboot after the installation completes, Kali would automagically connect back to the attacker using a reverse OpenVPN connection. The VPN setup would then allow the attacker to bridge the remote and local networks as well as have access to a full suite of penetration testing tools on the target network.

Cool concept right? A ISO that you could fire and forget in a network that would allow the pentesting team to access the network with minimal involment from the System Administrator or Network Administration team. It would do this with an OpenVPN client, that would call back to the OpenVPN server that attacker/pentester controls. Closely resembling a reverse shell. The article was made in 2013 which is a little dated (Currently 2022), but luckly, Offensive Security has [another article](https://www.kali.org/docs/development/dojo-mastering-live-build/) that was "Updated" Jan 2022. This article turned out to be inaccurate, outdated, or just plain wrong.

## Knowledge, Terms, and commands

Make Folder and parent folders if they dont exist. If the folders do exists, no error is returned and the command contiunes. 
```bash
mkdir -p /path/to/folder
```


