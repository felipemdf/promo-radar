# promo-radar

# **Arquitetura do Sistema – Promo Radar**

Este documento descreve a arquitetura geral do sistema composto por três componentes principais:
**Aplicação Principal**, **Scraper** e **Aplicação de IA**.

---

# **1. Visão Geral da Arquitetura**

A arquitetura é composta por três serviços independentes, comunicando-se via HTTP e Mensageria:

* **Aplicação Principal (Web App)**
  Área do Cliente e Área do Admin em uma única aplicação (JSF ou Thymeleaf).
  Responsável pela gestão e exibição das promoções.

* **Scraper**
  Rotina dedicada a coletar posts, imagens e textos brutos de mercados.
  Não armazena nada no banco.
  Apenas envia os dados brutos para a Aplicação de IA.

* **Aplicação de IA**
  Serviço responsável por extrair informações estruturadas das imagens.
  Publica o resultado via Broker para a Aplicação Principal.
  Possui seu próprio banco (DB IA) com versões de prompts e logs.

---

# **2. Arquitetura Lógica (com Comunicações)**

```
                     +-------------------------------------+
                     |           Aplicação Principal        |
                     |      (Admin + Cliente Web App)       |
                     +-------------------+-------------------+
                                         ^
                                         | Consome eventos
                                         | (Promoções extraídas)
                                         |
                            +------------+-----------+
                            |    Broker de Mensagens |
                            | (RabbitMQ / Kafka / SQS)|
                            +------------+-----------+
                                         ^
                                         |
                                         | Publica evento:
                                         | { promo_extracted }
                                         |
+---------------------+       +----------+-----------+
|       Scraper       |  HTTP |      Aplicação de IA |
| (Rotina Coletora)   +------>+ - Endpoint: /extract |
| - Baixa imagens     | POST  | - Executa IA         |
| - Envia imagem      |       | - Salva logs / prompt|
+---------------------+       | - Publica no broker  |
                              +-----------------------+
```

---

# **3. Comunicação Entre os Sistemas**

### **Scraper → Aplicação de IA**

* **Tipo:** HTTP REST
* **Endpoint:** `POST /extract`
* **Dados enviáveis:**

  * URL da imagem
  * base64 (opcional)
  * texto bruto (opcional)

### **Aplicação de IA → Broker**

* **Tipo:** Mensageria
* **Ação:** Publica JSON da promoção extraída
* **Formato do evento:** `promotion_extracted`

### **Aplicação Principal → Broker**

* **Ação:** Consome eventos e salva no DB Principal

### **Aplicação Principal ↔ Aplicação de IA (Opcional)**

Apenas para diagnósticos ou afinalização.
Comunicação via HTTP opcional.

---

# **4. Diagrama de Sequência Completo**

```
Scraper           Aplicação de IA             Broker               Aplicação Principal
   |                     |                      |                       |
   |---POST /extract----->|                      |                       |
   |        (imagem)      |---Executa IA-------->|                       |
   |                      |---Publica----------->|                       |
   |                      |                      |---Entrega----------->|
   |                      |                      |      Promoção         |
   |<-------200-----------|                      |                       |
```

---

# **5. Responsabilidades dos Sistemas**

### **5.1. Aplicação Principal**

* Admin + cliente no mesmo sistema.
* Exibe promoções, permite organização por carrinho.
* Recebe dados prontos da IA.
* Salva no banco principal.
* Permite correções de promoções com erros.

### **5.2. Scraper**

* Roda periodicamente (cron, scheduler ou Lambda).
* Coleta imagem do post.
* Extrai URL ou baixa imagem.
* Envia para a Aplicação de IA.
* Não salva banco.
* Não processa IA.

### **5.3. Aplicação de IA**

* Exponde endpoint `/extract` (receber dados crus).
* Executa modelo IA com prompt configurado.
* Salva:

  * versões de prompt,
  * logs mínimos,
  * registros de input.
* Publica promoção estruturada no broker.
* É responsável por validar, melhorar ou versionar a extração.

---

# **6. Bancos de Dados**

### **DB Principal**

* Produtos
* Promoções
* Correções humanas
* Auditoria do Admin

### **DB IA**

* Versions de prompt
* Config do modelo
* Logs mínimos (somente referencia da imagem + json extraído)
* Nenhum JSON corrigido do Admin (isso fica no DB Principal)

---

# **7. Fluxo Completo do Sistema**

1. Scraper captura a imagem/post.
2. Envia para `/extract` na Aplicação de IA.
3. IA processa a imagem.
4. IA publica evento `promotion_extracted` no broker.
5. Aplicação Principal consome o evento.
6. Salva a promoção no DB Principal.
7. Admin corrige informações quando necessário.
8. Correções podem futuramente treinar novas versões de prompt.
