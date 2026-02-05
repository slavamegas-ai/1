# Security Checklist (VPS + On-prem Docker + n8n + Postgres + Ollama)

Цель: минимальный, но жёсткий набор проверок, без которых нельзя считать инфраструктуру безопасной.[file:3]  

---

## 1. VPS: Firewall (UFW)

**Цель:** ограничить доступ к SSH и Postgres только с доверенных IP.

### Шаги

1. Включить UFW и базовые правила:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow from <VPN_IP> to any port 5432 proto tcp
sudo ufw enable
sudo ufw status verbose
