# Automação de E-mails com Relatórios Diários

Este projeto automatiza o envio de e-mails com relatórios em anexo, lidando com formatação HTML, assinaturas embutidas, anexos binários e regras de negócio para dias úteis (ignorando finais de semana).

## 1. Módulos e Bibliotecas (Imports)

O script utiliza uma série de bibliotecas nativas e externas para garantir que o e-mail chegue perfeitamente ao destinatário.

| Biblioteca/Módulo | O que faz no script |
| :--- | :--- |
| `os` | Acessa os diretórios e verifica a existência dos arquivos na máquina local. |
| `datetime` | Gera as datas corretas para puxar os arquivos de relatório atualizados. |
| `dotenv` | Carrega o arquivo `.env` para ocultar senhas e credenciais do código-fonte. |
| `smtplib` | Protocolo padrão SMTP. É o carteiro: faz o login no servidor do Google e aperta o botão "enviar". |
| `MIMEMultipart` | O "pacote" principal do e-mail. Junta corpo, anexos e imagens para que tudo seja enviado como uma coisa só. |
| `MIMEText` | Formata o corpo do e-mail (código HTML) para ficar visualmente amigável dentro do pacote Multipart. |
| `MIMEBase` | Cria um espaço seguro para acoplar arquivos que não são de texto bruto (arquivos binários). |
| `encoders` | Converte arquivos binários em Base64. Impede que o arquivo chegue corrompido ou quebrado do outro lado. |
| `Header` / `encode_rfc2231` | Traduz nomes de arquivos com acentos (ç, ã, é) para o padrão global da internet. Evita que o e-mail seja barrado por caracteres ilegíveis. |

## 2. Lógica de Datas (O Ajuste da Segunda-feira)

Para economizar tempo e evitar trabalho manual no início da semana, o script possui uma lógica inteligente para dias úteis.
* Quando o sistema detecta que o dia atual é segunda-feira (`.weekday() == 0`), a data alvo subtrai 3 dias (`day - 3`), buscando os dados da sexta-feira passada.
* Em qualquer outro dia da semana, o script apenas subtrai 1 dia (`day - 1`).

## 3. Construção do E-mail (Função Principal)

A função principal responsável pelo envio recebe remetente, senha, destinatários, assuntos e caminhos dos arquivos. O processo de montagem segue estes passos:

* **Agrupamento (`msg = MIMEMultipart()`):** Instancia o contêiner do e-mail. O campo "Para" (To) e "Com Cópia" (Cc) usam `", ".join(destinatarios)` para suportar múltiplas pessoas.
* **Corpo do E-mail:** Anexa o HTML usando `msg.attach(MIMEText(html_body, 'html'))`.
* **Assinatura Inline:** A imagem da assinatura recebe um ID de conteúdo (`Content-ID`, ex: `<assinatura_email>`). Quando o Outlook abre o e-mail e lê o HTML com `src="cid:assinatura_email"`, ele renderiza a imagem que foi anexada junto com a mensagem.
* **Processamento de Anexos:**
  * O código verifica se o arquivo existe usando `os.path.exists()`.
  * Se existir, usa `MIMEBase('application', 'octet-stream')` para sinalizar que é uma sequência de bytes.
  * O `encoders.encode_base64` prepara o arquivo para transporte seguro.
  * O `Header` formata o nome para evitar bugs com caracteres especiais.
  * Uma contagem valida o sucesso. Se nenhum anexo for encontrado, a operação é abortada (`return False`) antes do envio.

## 4. O Servidor SMTP

A etapa final liga o servidor de envio padrão:
1. Conecta via `smtplib.SMTP` (porta do Google).
2. Inicia a criptografia com `starttls()`.
3. Faz o login com credenciais seguras.
4. Envia o pacote completo com `sendmail()`.
5. Encerra a conexão com `server.quit()`.

Caso ocorra qualquer erro no processo, o script printa a falha e retorna `False` para pausar aquele envio específico.

## 5. Dicionário de Destinatários e Execução

As regras de quem recebe o quê estão mapeadas no bloco `DESTINATARIOS`. Ali estão definidos: nomes, arquivos específicos, CCs e o corpo de e-mail customizado.

No laço de execução principal, o script:
* Itera sobre a lista de e-mails a enviar.
* Cria uma lista temporária (`lista_atual_anexos`) usando `.append()` para juntar arquivos quando um destinatário precisa receber mais de um relatório.
* Chama a função `enviar_email_relatorio`, passando as variáveis em ordem (remetente, destinatários, assunto, etc.) e o caminho da assinatura.
* O loop se repete até que todos os e-mails mapeados na configuração sejam disparados.
