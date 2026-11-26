# 13. Referência (msfvenom, tmux, bash)

### tmux
```text
tmux new -s <nome>      # nova sessão      | tmux ls        # listar
Ctrl+b c                # nova janela      | tmux a -t <nome>  # attach
Ctrl+b d                # detach           | Ctrl+b z       # zoom no pane
Ctrl+b espaço           # trocar layout
```

### atalhos do bash
```text
Ctrl+a   início da linha       Ctrl+e   fim da linha
Alt+.    navega histórico de palavras
```

### Metasploit handler (rápido)
```text
use exploit/multi/handler; set PAYLOAD <payload>; set LHOST <ip>; set LPORT <port>; run
```