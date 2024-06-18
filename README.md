[![Logo](https://inlocsistemas.com.br/wp-content/uploads/2021/04/logo.png)](https://inlocsistemas.com.br)

# Documentação do webApp pedidoApp

O pedidoApp é um sistema web voltado para gestão de pedidos e pagamentos, ideal para estabelecimentos como bares e restaurantes. Ele oferece diversas funcionalidades para facilitar o processo de pedido e pagamento.

## Funcionalidades

- **Pedido de Produtos:** Permite que clientes façam pedidos de produtos diretamente pelo aplicativo, facilitando o processo de escolha e solicitação.

- **Compra de Créditos:** Funcionalidade que permite aos usuários comprarem créditos pré-pagos, proporcionando conveniência e agilidade no processo de pagamento.

- **Pagamento de Compras em Aberto (Pós-pago):** Facilita o pagamento de contas pós-pagas, proporcionando flexibilidade aos clientes e estabelecimentos.

- **Utilização por Garçom:** Pode ser utilizado por garçons para realizar pedidos, facilitando o atendimento e melhorando a experiência do cliente.

- **Sistema de Pagamento Integrado com API da Cielo:** Integração com a API de pagamento da Cielo, permitindo pagamentos seguros e eficientes através de diversas modalidades como PIX, cartão de crédito e débito.

## Benefícios

- **Facilidade de Uso:** Interface intuitiva que simplifica o processo de pedidos e pagamentos.

- **Segurança:** Pagamentos seguros com integração robusta com a API da Cielo.

- **Flexibilidade:** Adaptável para diferentes tipos de estabelecimentos, como bares e restaurantes.

Este documento serve como guia inicial para entender as principais funcionalidades e benefícios do pedidoApp. Para mais detalhes sobre como configurar e utilizar o sistema, consulte a documentação completa disponível no repositório `pedidoApp-docs`. Para mais informações visite nosso site [Inloc Sistemas.](https://inlocsistemas.com.br/) 

# Regras de Negócio do Sistema

Para que um pedido seja concluído com sucesso, as seguintes regras de negócio devem ser seguidas:

## Regras de Negócio - Pedido

- Todos os campos na tela devem estar preenchidos.

- O **caixa do operador no WebApp deve estar aberto**; caso contrário, a API retornará (`LockedException`, caixa fechado).

- Deve ser um **sócio válido e não bloqueado**; se não for, a API retornará (`LockedException`, sócio bloqueado).

- O **ponto de entrega deve ser informado**; caso contrário, a API retornará (`BadRequestException`, Ponto de entrega deve ser informado).

- O **pedido não deve conter produtos bloqueados**; caso contrário, a API retornará (`BadRequestException`, Não foi possível finalizar a venda, pois o produto xpto está bloqueado!).

- O **pedido não deve conter produtos que estejam fora do período de produção**; caso contrário, a API retornará (`BadRequestException`, Produto xpto não está no período de produção!).

- O **pedido não deve conter produtos que estejam em um ponto de produção inativo**; caso contrário, a API retornará (`BadRequestException`, Produto xpto não possui um ponto de produção ativo no momento).

- O **pedido não deve conter itens se o ponto de produção estiver configurado para emitir tickets**; caso contrário, a API retornará (`BadRequestException`, Ponto de produção xpto não emite comandas).

### Diagrama de sequência
![Diagrama de sequência](regras-de-negócio-do-sistema.webp)

# Documentação APIs da Cielo - Cartão

## Fluxo pedido com Cartão de débito
Criação de um pedido e cobrança no cartão de débito.
- Todos os campos devem ser preechidos

- **Para criar um pedido e fazer a cobrança em um cartão de débito e nessesacio fazer autenticação 3DS seguindo o seguinte fluxo**

- O metodo prepareProcessDebitCard e chamado para iniciar a requisição do token3ds 

```java
	public String prepareProcessDebitCard() {
		try {
			// Outros métodos
			if (this.validate()) {
				this.debitCardToken = this.paymentCardService.create3DSToken(); // 
				this.preparedDebitCard = true;
			}
			return "";
		} catch (Exception e) {
			e.printStackTrace(); // GERAR LOG
			showErrorMessage("Erro", "Erro ao finalizar pedido, tente novamente!");
		}
		return "";
	}
```  
Para criação do 3ds-token e chamado o end point @get/v1/payment-cards/3ds-token no pedidoAppApi que por sua ver faz a requisição no  pagamentosApiCIELO @get/v1/card/3ds-token
solicitando um token a cielo no @post/v2/auth/token para autenticar a transação. 

- Apos a criação do 3ds-token e chamado api de autenticação da cielo para validar os dados atraves do [Script cielo.](https://developercielo.github.io/manual/3ds#integra%C3%A7%C3%A3o-do-script)

- Com os dados do pedido e chamado o createOrderWithCard para iniciar o processo de criação e cobrança do pedido
```java
	public String finalizeDebitCard() {
		try {
			this.preparedDebitCard = false;

			String cavv = null;
			String xid = null;
			String eci = null;
			String version = null;
			String referenceId = null;

			Map<String, String> requestParamMap = FacesContext.getCurrentInstance().getExternalContext()
					.getRequestParameterMap();

			if (requestParamMap.containsKey("cavv")) {
				cavv = requestParamMap.get("cavv");
			} else {
				throw new Exception("Cavv não retornado!");
			}
			if (requestParamMap.containsKey("xid")) {
				xid = requestParamMap.get("xid");
			} else {
				throw new Exception("Xid não retornado!");
			}
			if (requestParamMap.containsKey("eci")) {
				eci = requestParamMap.get("eci");
			} else {
				throw new Exception("Eci não retornado!");
			}
			if (requestParamMap.containsKey("version")) {
				version = requestParamMap.get("version");
			} else {
				throw new Exception("Version não retornado!");
			}
			if (requestParamMap.containsKey("referenceId")) {
				referenceId = requestParamMap.get("referenceId");
			} else {
				throw new Exception("ReferenceId não retornado!");
			}

			this.orderService.createOrderWithCard(CheckOrderController.getBean().getOrder(), this.deliveryPointId,
					this.paymentCardSelected, this.cvv, cavv, xid, eci, version, referenceId, this.saveNewCard);

			CheckOrderController.getBean().clearOrder();
			showInfoMessage("Sucesso", "Pedido finalizado com sucesso!");
			return MenuController.URL;
		} catch (ClientException e) {
			if (e.getDetails() != null && !e.getDetails().isEmpty()) {
				e.getDetails().stream().forEach(it -> {
					showErrorMessage("Erro", it);
				});
			} else {
				showErrorMessage("Erro", e.getMessage());
			}
		} catch (Exception e) {
			e.printStackTrace(); // GERAR LOG
			showErrorMessage("Erro", "Erro ao finalizar pedido, tente novamente!");
		}
		return "";
	}

```
- O createOrderWithCard chama o client que faz uma chamada no pedidoappapi @post/v1/orders/card/ que inicializa o processamento do pedido 

-Detro do processamento do pedido o método processCieloTransactionCard faz a chamada do create(saleRequestDTO) que faz uma request na pagamentosApiCIELO @post/v1/payment/pay que por sua vez faz a requisição para cielo no @post/1/sales obtendo SaleResponseDTO da cielo.      
Com o sales response e processado a resposta do pagamento.

```java
    processCieloTransactionCard(
    // Outros métodos
    try {
           saleResponse = this.paymentApiCieloClient.create(saleRequestDTO);
		} catch (Exception e) {
			LOGGER.error("Erro ao conectar a api de pagamentos! " + e.getMessage()); 
			throw new InternalServerException("Erro ao conectar a api de pagamentos! ");
		}
        if (saleResponse.getPayment().getStatus() == 2) {
             // Outros métodos
            }
        } else if (saleResponse.getPayment().getStatus() == 3) {
            throw new UnauthorizedException("Pagamento negado por Autorizador!");
        } else if (saleResponse.getPayment().getStatus() == 13) {
            throw new UnprocessableEntityException("Pagamento cancelado por falha no processamento ou por ação do Antifraude!");
        } else {
            throw new InternalServerException("Erro ao processar pagamento!");
        }
   // Outros métodos
   
```

### Diagrama de sequência Débito
![Diagrama de sequência Débito](fluxo_pedido_app_cartao_debito.png)

## Fluxo pedido com cartão de crédito
Criação de um pedido e cobrança no cartão de crédito.
- Diagrama de sequência (Crédito)


# Documentação APIs da Cielo - Pix
Criação de um pedido e cobrança no Pix.
- Diagrama de sequência (Pix)
