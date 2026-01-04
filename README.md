# Projekt: VM z usługą WWW + podstawowe zabezpieczenia (AWS) — Event Planner (demo)

## Cel
Uruchomienie maszyny wirtualnej (EC2) z usługą WWW (Nginx) oraz podstawowe zabezpieczenie dostępu i sieci.

## Architektura (3.0)
- AWS IAM: dedykowany użytkownik `Eventplanner_user` (bez używania konta root do prac)
- AWS EC2: `eventplanner_vm` (Ubuntu)
- Security Group: `eventplanner_sg` (HTTP/HTTPS publicznie, SSH tylko z mojego IP)
- Usługa: Nginx serwujący stronę demo aplikacji “Event Planner”

## 1. Zarządzanie dostępem (IAM / least privilege)
1. Włączono MFA na koncie root.
2. Utworzono użytkownika IAM `Eventplanner_user`.
3. Do użytkownika przypięto politykę uprawnień (np. AmazonEC2FullAccess) w celu zarządzania EC2.
4. Dalsze prace wykonywano na koncie IAM (nie root).

**Dowody (screeny):**
- `screens/iam-user.png` — użytkownik IAM + permissions
- `screens/root-mfa.png` — potwierdzenie MFA na root

## 2. Bezpieczeństwo sieci (Security Group)
Inbound rules (eventplanner_sg):
- HTTP 80: 0.0.0.0/0
- HTTPS 443: 0.0.0.0/0
- SSH 22: 178.214.26.2/32

**Dowody (screeny):**
- `screens/security-group-inbound.png` — reguły inbound z widocznym SSH tylko z mojego IP

## 3. Wdrożenie usługi WWW (Nginx)
1. Połączenie SSH do VM:
   - `ssh -i eventplanner-key.pem ubuntu@16.171.11.56`
2. Instalacja i uruchomienie Nginx:
   - `sudo apt update && sudo apt -y upgrade`
   - `sudo apt -y install nginx`
   - `sudo systemctl enable --now nginx`
3. Wgranie strony demo do `/var/www/html/index.html`
4. Weryfikacja:
   - otwarcie `http://16.171.11.56` w przeglądarce.

**Dowody (screeny):**
- `screens/web-working.png` — działająca strona “Event Planner (demo)”
- `screens/ssh-connected.png` — terminal z połączeniem SSH (bez wrażliwych danych)

## Podstawowe hardening na VM
- UFW: zezwolono tylko na OpenSSH i Nginx Full

**Dowody (screeny):**
- `screens/ufw-status.png` — wynik `sudo ufw status verbose`


---

## Architektura (3.5) — segmentacja sieci

W kolejnym etapie projektu architektura została rozszerzona o segmentację sieci oraz pośrednie warstwy dostępu, zgodnie z dobrymi praktykami bezpieczeństwa chmurowego.

### Zastosowane elementy:
- Dedykowana sieć **VPC** (`eventplanner-vpc`)
- **Public subnet** — zasoby dostępne z internetu
- **Private subnet** — serwer aplikacyjny bez publicznego adresu IP
- **Bastion Host** — pośredni dostęp administracyjny (SSH)
- **NAT Gateway** — kontrolowany dostęp wychodzący do internetu
- **Application Load Balancer (ALB)** — dostęp do aplikacji WWW
- **Amazon CloudWatch** — monitoring zasobów

---

## 4. Segmentacja sieci (VPC, Subnets, Route Tables)

Utworzono dedykowaną sieć VPC:
- CIDR: `10.0.0.0/16`

### Subnety:
- **public-subnet** (`10.0.100.0/24`) — Load Balancer, Bastion Host
- **public-subnet-2** (`10.0.110.0/24`) — drugi subnet (inna AZ) dla ALB
- **private-subnet** (`10.0.200.0/24`) — serwer aplikacyjny

### Routing:
- **Public Route Table**:
  - `0.0.0.0/0 → Internet Gateway`
- **Private Route Table**:
  - `0.0.0.0/0 → NAT Gateway`

Serwer aplikacyjny nie posiada bezpośredniego dostępu do internetu ani publicznego adresu IP.

**Dowody (screeny):**
- `screens/vpc-subnets.png` — VPC i subnety
- `screens/private-rt.png` — tablice routingu (IGW / NAT)
- `screens/public-rt.png` — tablice routingu (IGW / NAT)

---

## 5. Dostęp administracyjny — Bastion Host

Dostęp SSH do serwera aplikacyjnego realizowany jest wyłącznie poprzez **Bastion Host** umieszczony w publicznej podsieci.

### Zasady:
- Bastion Host:
  - Publiczny adres IP
  - SSH (22) dostępne tylko z mojego adresu IP
- App Server:
  - Brak publicznego IP
  - SSH (22) dozwolone wyłącznie z `bastion-sg`

**Dowody (screeny):**
- `screens/bastion-instance.png` — bastion host z publicznym IP
- `screens/ssh-bastion-to-app.png` — połączenie SSH do prywatnej instancji

---

## 6. Dostęp do aplikacji WWW — Application Load Balancer

Dostęp do aplikacji webowej realizowany jest przez **Application Load Balancer (ALB)**.

### Konfiguracja:
- ALB:
  - Typ: Internet-facing
  - Subnety: `public-subnet`, `public-subnet-2`
  - Security Group: `alb-sg`
- Target Group:
  - Typ: Instance
  - Protokół: HTTP (80)
  - Zarejestrowana instancja: `eventplanner-app`
- App Server:
  - HTTP (80) dozwolone wyłącznie z `alb-sg`

Dostęp do aplikacji:
http://eventplanner-lb-72600078.eu-north-1.elb.amazonaws.com/


**Dowody (screeny):**
- `screens/alb-active.png` — ALB (status: Active)
- `screens/target-group-healthy.png` — Target Group (Healthy)
- `screens/app-through-alb.png` — działająca aplikacja przez DNS ALB

---

## 7. Serwer aplikacyjny (Private Subnet)

Serwer aplikacyjny:
- znajduje się w **private subnet**
- nie posiada publicznego adresu IP
- posiada dostęp wychodzący do internetu wyłącznie przez **NAT Gateway**

Na serwerze uruchomiono usługę **Nginx**, serwującą demo aplikacji „Event Planner”.

**Dowody (screeny):**
- `screens/app-instance-private.png` — brak public IP
- `screens/nginx-running.png` — status nginx

---

## 8. Monitoring (CloudWatch)

Dla serwera aplikacyjnego włączono monitoring w usłudze **Amazon CloudWatch**.

Monitorowane metryki:
- CPU Utilization
- Network In / Network Out

**Dowody (screeny):**
- `screens/monitoring.png`


---


