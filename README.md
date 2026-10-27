# NexusTech ERP + Loja

Aplicação PHP 8.2+ com loja e ERP no mesmo banco MySQL/MariaDB. A loja é pública; toda rota e todo endpoint administrativo verifica o perfil atual no banco, e uma conta `cliente` recebe `403` ao tentar acessá-los. O estoque só muda dentro de transações que também registram o pedido, a compra ou o motivo do ajuste.

## Estrutura

```text
index.php                 Front controller
routes.php                Rotas web e /api/v1
app/                      Controllers, serviços, PDO, middleware e helpers
api/helpers/response.php  Envelope JSON padronizado
admin/gerar_pdf.php       Geração do relatório Dompdf, incluída após RBAC
view/                     Templates PHP da loja, conta, pedidos e ERP
assets/css/               Design system responsivo
assets/js/                Módulos ES para loja, conta, checkout e ERP
database/                 Migrations SQL numeradas e dados do catálogo
tools/                    Migrations e seeds locais
storage/cache/            Cache privado das consultas analíticas
```

## Requisitos

- PHP 8.2 ou superior com `pdo_mysql` e `mbstring` habilitados.
- MySQL 8+ ou MariaDB compatível com InnoDB, JSON e `SELECT ... FOR UPDATE`.
- Composer 2.
- Apache com `mod_rewrite` no XAMPP ou PHP built-in server.

## Instalação com XAMPP

1. Inicie Apache e MySQL pelo painel do XAMPP; confirme que `pdo_mysql` e `mbstring` estão habilitados no `php.ini` usado pelo Apache e pelo CLI.
2. No diretório do projeto, copie `.env.example` para `.env` e ajuste `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER` e `DB_PASS` conforme o MySQL local, `APP_BASE_PATH=/NexusTech` e `APP_URL` conforme a porta do Apache; configure também um `WEBHOOK_SECRET` aleatório. Troque as duas senhas de demonstração antes de criar usuários.
3. Instale as dependências registradas no lockfile: `composer install`.
4. Crie o banco vazio configurado: `php tools/create-database.php`. O script usa `CREATE DATABASE IF NOT EXISTS` e não altera outros bancos.
5. Teste a conexão antes das migrations: `php tools/test-connection.php`.
6. Aplique as migrations: `php tools/migrate.php`.
7. Crie as contas locais de demonstração: `php tools/seed-users.php`.
8. A migration 006 cadastra 45 produtos fictícios adicionais e avaliações claramente identificadas como exemplos; junto dos cinco produtos iniciais, a vitrine passa a ter cerca de 50 itens. Essas avaliações não são de clientes reais nem substituem as avaliações verificadas.
9. Opcionalmente, popule as análises com 30 pedidos distribuídos no último mês: `php tools/seed-demo-orders.php`.
10. Abra `http://localhost:8080/NexusTech/`.

Em bancos já instalados, execute o migrador novamente para aplicar as migrations pendentes `004_auth_version.sql`, `005_erp_expansion.sql` e `006_demo_catalog.sql`. A migration 004 invalida sessões existentes na primeira requisição após a atualização; usuários deverão entrar novamente. A migration 005 cria os saldos por armazém, lotes/séries, transferências, inventário físico e contas financeiras, além de conciliar os dados existentes. A migration 006 adiciona o catálogo demonstrativo.

As credenciais locais são as definidas por `SEED_ADMIN_EMAIL`, `SEED_ADMIN_PASSWORD`, `SEED_CUSTOMER_EMAIL` e `SEED_CUSTOMER_PASSWORD` no `.env`. O script de contas recusa senhas com menos de 12 caracteres e só executa quando `APP_ENV=local`. Não use essas contas/senhas em produção.

## Servidor PHP local

Ajuste `APP_BASE_PATH=` (vazio) e `APP_URL=http://127.0.0.1:8000`, rode `composer serve` e acesse essa URL. O roteador do servidor local publica apenas assets; arquivos internos, migrations e scripts utilitários não são servidos diretamente.

## Fluxos para validar

- Loja: busque por nome/SKU, filtre categoria/preço/disponibilidade, abra um produto, navegue pela galeria, adicione ao carrinho e confirme a persistência ao recarregar. Clientes podem avaliar produtos depois de um pedido entregue; cada cliente tem uma avaliação por produto, que pode ser atualizada.
- Catálogo acadêmico: o banco fica com aproximadamente 50 produtos fictícios. Cards mostram trechos da descrição e notas agregadas; a página de detalhes apresenta a biografia completa e avaliações de demonstração explicitamente identificadas. Nenhum desses dados é apresentado como compra ou opinião real.
- Conta cliente: cadastre-se ou entre, atualize perfil/endereço, faça checkout e veja a linha do tempo em **Meus pedidos**.
- Consistência: tente comprar mais unidades que o estoque. A API deve rejeitar sem criar pedido nem movimentação parcial. Cancele pedido pendente e valide a reposição no ERP.
- RBAC: entre como cliente e solicite `/admin` e `/api/v1/admin/dashboard`; deve receber `403` sem dados administrativos. Desative uma conta já conectada e valide que a próxima chamada autenticada encerra a sessão. Entre como admin e confirme acesso ao painel, à loja e ao próprio perfil/endereço.
- ERP: entre com a conta admin; use a navegação do painel para acessar estoque, pedidos, compras, fornecedores, usuários, categorias, armazéns, lotes/séries, inventário físico, financeiro e insights. Ajustes, recebimentos, vendas, cancelamentos e transferências sincronizam o saldo por armazém com o total do produto. Inventários em andamento bloqueiam movimentações no local, e cada contagem precisa ser preenchida antes de ser aplicada.
- Financeiro/insights: contas a pagar de compras e a receber de pedidos são geradas junto das operações; contas manuais podem ser registradas/baixadas. Insights são alertas determinísticos de estoque, lotes, vendas e vencimentos, não uma integração de IA generativa.
- Análises: abra o dashboard, teste hoje/7/30/90 dias e datas personalizadas, exporte gráficos PNG, listas CSV e o relatório PDF.
- Checkout/pagamento: o checkout valida o endereço por etapas e busca preço/estoque atuais antes da confirmação. Pix, cartão e boleto são simulados; não são coletados dados de cartão. O frete também é simulado (R$ 19,90 ou grátis acima de R$ 500). Pedidos permanecem pendentes até atualização administrativa ou webhook assinado.

O webhook `POST /api/v1/webhooks/pagamento` valida `X-Webhook-Timestamp` e `X-Webhook-Signature`. A assinatura é `HMAC-SHA256(timestamp + "." + corpo JSON, WEBHOOK_SECRET)`; timestamps fora de cinco minutos e `event_id` repetidos são rejeitados/ignorados.

## API principal

- Pública: `GET /api/v1/produtos`, `GET /api/v1/produtos/{id}`, `GET /api/v1/categorias`.
- Avaliações: `GET /api/v1/produtos/{id}/avaliacoes` é público; `POST` exige sessão/CSRF e um pedido entregue com o produto.
- Sessão: `GET /api/v1/csrf`, `POST /api/v1/auth/register`, `POST /api/v1/auth/login`, `POST /api/v1/auth/logout`, recuperação e redefinição de senha.
- Cliente: `GET/PATCH /api/v1/conta`, `GET/POST/DELETE /api/v1/enderecos`, `GET/POST /api/v1/pedidos`, cancelamento e validação de cupom.
- Administrativa: `/api/v1/admin/*`, sempre limitada a `role=admin` no servidor.

Respostas JSON seguem `success/data/meta` ou `success/error`. Escritas por sessão exigem `X-CSRF-Token`; o cliente ES module busca o token em `/api/v1/csrf`.

## Notas operacionais

- A recuperação envia e-mail pela função `mail()` do PHP. Configure SMTP/sendmail no XAMPP ou substitua o transporte para seu provedor antes de produção.
- Chart.js e a tipografia são carregados de CDNs definidos na Content Security Policy. Para operação sem internet, sirva essas bibliotecas localmente.
- Os pagamentos e o cálculo de frete são deliberadamente simulados; nenhum dado de cartão real é coletado. Integre um gateway real antes de aceitar pagamentos.
- As migrations SQL do MySQL fazem commits implícitos em DDL. Se uma migration falhar, corrija o ambiente e reaplique; o migrador reconhece colunas já criadas nas operações `ALTER TABLE ... ADD COLUMN`, e as tabelas/seeds usam operações idempotentes.
- O arquivo `database/schema.sql` é uma instalação consolidada para bancos vazios; em banco existente, use apenas `php tools/migrate.php` para aplicar migrations pendentes.
- Antes de publicar, configure HTTPS, `APP_DEBUG=false`, segredos fortes, transporte de e-mail, backups, rotação/limpeza de logs e políticas de retenção. Não use as contas/senhas de demonstração em produção nem o servidor embutido do PHP.
- O `.htaccess` encaminha as rotas ao front controller, bloqueia diretórios internos e arquivos de configuração e impede a execução HTTP direta de scripts PHP. O Apache precisa permitir `AllowOverride All` e ter `mod_rewrite` habilitado.
- Gere valores aleatórios distintos para `APP_KEY`, `SESSION_SECRET` e `WEBHOOK_SECRET`; os valores de exemplo são apenas placeholders. Pagamento real, frete, e-mail transacional e IA generativa ainda dependem de provedores e credenciais e não estão integrados.
- Administradores autenticados também podem abrir perfil, endereços, pedidos e checkout; clientes continuam limitados às rotas de loja e conta, e as rotas `/admin` seguem protegidas por `role=admin`.
