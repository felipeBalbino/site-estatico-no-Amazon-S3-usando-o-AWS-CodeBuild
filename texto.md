Links:

https://www.karelbemelmans.com/2017/01/deploying-a-hugo-website-to-amazon-s3-using-aws-codebuild/
https://blog.cloudboost.io/automated-deployment-of-serverless-and-react-using-aws-codepipeline-e268bbb032e
https://medium.com/@hzburki.hzb/automate-static-site-deployment-on-s3-with-aws-codebuild-8b2546a360df
https://gist.github.com/bradwestfall/b5b0e450015dbc9b4e56e5f398df48ff

# S3 Sites estáticos

O que isso cobrirá
- Hospede um site estático no S3
- Redirecione `www.website.com` para` website.com`
- O site pode ser um SPA (exigindo que todas as solicitações retornem `index.html`)
- Certificados SSL gratuitos da AWS
- Implantação com invalidação de CDN

## Recursos
- https://stormpath.com/blog/ultimate-guide-deploying-static-site-aws
- https://miketabor.com/host-static-website-using-aws-s3/
- http://www.mycowsworld.com/blog/2013/07/29/setting-up-a-godaddy-domain-name-with-amazon-web-services.html
- https://www.davidbaumgold.com/tutorials/host-static-site-aws-s3-cloudfront/#make-an-s3-bucket

## S3 Bucket

- Crie um bucket S3 com o nome exato do nome do domínio, por exemplo, `website.com`. 
- Em ** Propriedades **, clique na seção Site estático.
  - Clique em ** Use este intervalo para hospedar um site ** e insira `index.html` no campo ** Index Document **.
  - Não insira mais nada neste formulário.
  - Isso criará um "ponto final" na mesma tela semelhante a `http: // website.com.s3-website-us-east-1.amazonaws.com`.
- Em seguida, clique na guia ** Permissões ** e, em seguida, ** Política de bucket **. Digite esta política:

`` json
{
    "Versão": "17/10/2008",
    "Declaração": [
        {
            "Sid": "AllowPublicRead",
            "Efeito": "Permitir",
            "Diretor": {
                "AWS": "*"
            }
            "Ação": "s3: GetObject",
            "Recurso": "arn: aws: s3 ::: BUCKET_NAME / *"
        }
    ]
}
`` ``

> Certifique-se de substituir `BUCKET_NAME` pelo seu.

> Nota: A nomeação do bucket não precisa ser exatamente o nome do domínio. Eu li isso em vários artigos que precisava ser, mas não precisa. Se estiver usando domínios curinga na AWS, li que não podemos ter pontos no nome do domínio ao usar domínios curinga. Portanto, saiba que você pode nomear o intervalo de qualquer maneira, mas usar pontos funciona se não estiver usando domínios curinga

O upload de um `index.html` deve nos permitir visitar o" ponto final "

## CloudFront

- Vá para a seção CloudFront e clique em ** Criar distribuição ** e crie para ** Web **, não RTMP.
- Em ** Nome de Domínio de Origem **, cole o "ponto final" criado anteriormente no S3 (sem a parte `http: //`). Observe que, quando você clicar nesse campo, ele funcionará como um menu suspenso com opções para seus buckets existentes. Eu acho que você pode simplesmente selecionar um desses dois, que é uma lista válida dos seus buckets S3.
- A ordem destas instruções pressupõe que os certificados SSL ainda não estejam configurados. Portanto, não faça nada com as configurações relacionadas ao SSL
- Selecione "yes" para ** Compactar objetos automaticamente **.
- Em ** Nomes de Domínio Alternativos (CNAMEs) **, coloque os nomes de domínio que você deseja corresponder a este intervalo. Coloque cada um em sua própria linha ** OU ** separados por vírgula. A razão pela qual você pode ter dois ou mais é mais ou menos assim: `mywebsite.com` e` www.mywebsite.com`. O campo é chamado "Nomes de Domínio Alternativos" porque a AWS terá um nome de domínio específico do aws para a CDN, mas você não deseja usá-lo, por isso, deverá colocar seus domínios personalizados e usar o Route 53 (próximo seção) para apontar domínios para a CDN.
- Em ** Objeto raiz padrão **, digite `index.html`.
- Crio. A próxima tela mostrará distribuições em forma de tabela, a que acabamos de fazer estará "em andamento" por alguns minutos

A distribuição terá um nome de domínio como `dpo155j0y52ps.cloudfront.net`. Isso é importante para o DNS (veja abaixo). Então copie-o de alguma forma.
  
## Rota 53

Essas instruções de DNS assumem que seu DNS está hospedado na AWS. ** Isso não significa ** que você precisa comprar um domínio na AWS, apenas significa que quando você compra um domínio em algum lugar como Google ou GoDaddy, lá é necessário apontar os registros NS para a AWS para permitir que a AWS gerencie as peças do registro DNS. Mas primeiro, na AWS, é onde você cria a "Zona hospedada", onde é possível criar os valores de NS que eventualmente serão dados ao Google ou ao GoDaddy etc. Não sei como isso é diferente se você comprar seu domínio na AWS (Mas, novamente, nunca compro domínios no mesmo local que hospedo)

- Clique em ** Zonas hospedadas **
- Crie uma nova zona: use o nome do domínio (`mywebsite.com` sem subdomínio) para a zona. Observe que cada nome de domínio terá uma zona, todos os subdomínios pertencem à mesma zona.
- Isso deve criar registros NS, como:

`` ``
ns-1208.awsdns-23.org. 
ns-2016.awsdns-60.co.uk. 
ns-642.awsdns-16.net. 
ns-243.awsdns-30.com.
`` ``

- Os registros NS podem ser usados ​​para direcionar o gerenciamento de DNS de outro registrador de domínio para o AWS Route 53
- Clique em ** Criar conjunto de registros ** para criar um registro `A`.
  - Este será o registro que aponta `mywebsite.com 'para o CloudFront.
  - Para o ** nome **, não insira nenhum valor
  - Altere ** Alias ​​** para Sim
  - Cole o domínio CloutFront no campo ** Alias ​​**
    - Isso deve se parecer com `[some-random-number] .couldfront.net`. Você pode fazer isso clicando na distribuição do CloudFront e, na guia Geral, há um rótulo "Nome de domínio".
  - Clique em Criar conjunto de registros
- Crie outro registro `A` para o redirecionamento` www`
  - Siga as mesmas etapas para o registro anterior `A`, mas digite` www` para ** nome ** e use o mesmo domínio do CloudFront. Mas observe que isso ocorre porque queremos que `www.mywebsite.com` e` mywebsite.com` aponte para o mesmo intervalo (e, portanto, o mesmo domínio do CloudFront). Suponho que você criaria um novo balde e uma nova distrubuição do CloudFront (com um novo domínio CF) se quisesse um segundo projeto em `app.mywebsite.com '. Isso pode ser comum se o aplicativo for um aplicativo React que seja um código completamente separado do site da "página inicial", que pode ser de um gerador de site estático ou algo assim.

## HTTPS

No Console da AWS, acesse ** Certificate Manager ** e solicite um certificado para o domínio e todos os subdomínios. Seremos obrigados a verificar o certificado por email ou DNS. Ao verificar por e-mail, a AWS procurará as informações públicas do proprietário do DNS e usará até três e-mails que encontrar lá (se as informações de propriedade do domínio forem públicas). Mas, mesmo que não seja público, a AWS também os usará (que você não pode escolher)

- `administrador @ mywebsite.com`
- `hostmaster @ mywebsite.com`
- `postmaster @ mywebsite.com`
- `webmaster @ mywebsite.com`
- `admin @ mywebsite.com`

Se a sua empresa usa o "webmaster @", não se preocupe, pois seu aplicativo provavelmente tem 1000 anos.

** Para TLDs .io **: http://docs.aws.amazon.com/acm/latest/userguide/troubleshoot-iodomains.html

Se você optar por verificar via DNS, a AWS solicitará que você adicione alguns registros CNAME ao DNS do Route 53, mas o mais interessante é que existe um botão de atalho para fazê-lo (para cada domínio e subdomínio) no Gerenciador de Certificados seção.

Após a verificação e o certificado ser "emitido", podemos voltar ao CloudFont para editar nossa distribuição para este domínio:

- Clique na distribuição e, na próxima página (na guia Geral), clique em ** Editar **
- Marque a caixa para ** Certificado SSL personalizado **
- Selecione nosso certificado e salve. Observe que o que parece um campo de texto é realmente um menu suspenso quando você clica nele para escolher seu certificado
- Quando terminar o formulário, clique na guia ** Comportamentos ** e edite o único registro que deve estar lá
- Selecione ** Redirecionar HTTP para HTTPS **. Clique em Save

## SPA

Se o site for um SPA, precisamos garantir que todas as solicitações ao servidor (neste caso, S3) retornem algo, mesmo que não exista arquivo. Isso ocorre porque os SPAs, como o React (com o React Router), precisam da página `index.html` para todas as solicitações; depois, coisas como as páginas" não encontradas "são tratadas no front-end.

Vá para o CloudFront e clique na distribuição à qual deseja aplicar essas configurações do SPA. Clique na guia ** Páginas de erro ** e adicione uma nova página de erro. Preencha o formulário com estes campos:

- ** Código de erro HTTP **: 404
- ** TTL **: 0
- ** Resposta de erro personalizada **: Sim
- ** Caminho da página de resposta **: `/ index.html`
- ** Código de resposta HTTP **: 200

## Desdobramento, desenvolvimento

Para a implantação, precisamos considerar que os arquivos na CDN do CloudFront não devem mudar. Se formos fazer o upload de novos arquivos para o S3, eles não serão implantados nos servidores de borda da CDN e, portanto, não atualizarão o site. [Leia mais] (http://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/ReplacingObjectsSameName.html).

Para invalidar arquivos na CDN, precisamos usar o recurso ** invalidations ** do CloudFront: [Leia mais] (http://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Invalidation.html).

No console da AWS, no gerenciamento de uma distribuição do CloudFront, há uma guia para ** Invalidações **. Poderíamos criar manualmente uma invalidação (com o valor de `/ *`) para invalidar todos os arquivos S3. Observe que os registros de invalidação aqui são invalidações únicas e toda vez que implantarmos novos arquivos, precisaremos fazer uma nova invalidação.

Para implantar com invalidações, precisaremos [instalar o AWS-CLI] (https://aws.amazon.com/cli/) primeiro. Também supomos que você tenha um usuário IAM da AWS com uma chave de acesso e uma chave de acesso secreta.

Para testar a instalação, faça:

`` sh
aws --version
`` ``

Configure o aws-cli:

`` sh
aws configure --profile PICK_A_PROFILE_NAME
`` ``

> Observe que o uso de "perfis" para configurar a AWS-CLI é provavelmente o melhor, pois você pode usar a CLI para gerenciar várias contas da AWS em algum momento. Certifique-se de trocar `PICK_A_PROFILE_NAME` pela sua escolha de nome (pode ser qualquer coisa).

Digite estes valores:
`` ``
ID da chave de acesso da AWS [Nenhum]: [sua chave de acesso]
Chave de acesso secreta da AWS [Nenhuma]: [Sua chave de acesso secreta]
Nome da região padrão [Nenhum]: us-east-1
Formato de saída padrão [Nenhum]: json
`` ``

Isso salvará suas entradas em `~ / .aws / credenciais`. Observe que você precisa inserir sua região correta para o seu material da AWS. Eu usei `us-east-1`, mas certifique-se de usar o correto para você. Observe também que você pode ter respostas em `text` em vez de` json` se desejar

Você pode omitir as duas últimas perguntas para região e formato, se desejar configurar um padrão para o seu computador (que todos os perfis usarão). O perfil padrão está localizado em `~ / .aws / config`. Se você omitir a região e o formato do seu perfil, verifique se eles existem no seu `~ / .aws / config` como:

`` sh
[padrão]
output = json
região = us-leste-1
`` ``

Agora, como precisamos executar alguns comandos do CloudFront que são "experimentais", precisamos fazer:

`` sh
aws configure set preview.cloudfront true
`` ``

Isso resultará em mais registros em `~ / .aws / config`.

Devemos estar configurados agora para dest uma implantação. Corre:

`` sh
aws s3 sync --acl public-read --profile YOUR_PROFILE_NAME - exclua build / s3: // BUCKET_NAME
`` ``

- Obviamente, substitua `YOUR_PROFILE_NAME` e` BUCKET_NAME` pelo seu. Isso também assume que a pasta que você deseja enviar é `build '.
- Este comando será
  - Verifique se todos os novos arquivos enviados são públicos (`--acl public-read`)
  - Verifique se estamos usando suas credenciais em seu perfil local da AWS (`--profile YOUR_PROFILE_NAME`)
  - Remova todos os objetos S3 existentes que não existem localmente (`--delete`)

Após a implantação ser verificada e bem-sucedida, precisamos invalidar:

`` sh
aws cloudfront --profile YOUR_PROFILE_NAME create-invalidation - id da distribuição YOUR_DISTRIBUTION_ID - caminhos '/ *'
`` ``

- Obviamente, substitua `YOUR_PROFILE_NAME` e` YOUR_DISTRIBUTION_ID` pela sua. Observe que seu ID de distribuição pode ser encontrado na seção CloudFront do console da AWS.
- Se a invalidação funcionou, você poderá ver um registro na guia ** Invalidações ** depois de clicar em sua distribuição.

Para facilitar tudo, adicione ao `package.json`:

`` json
  "scripts": {
    "deploy": "aws s3 sync --acl public-read --profile XYZ - exclua build / s3: // XYX && npm run invalidate",
    "invalidate": "aws cloudfront --profile XYZ create-invalidation --distribution-id XYZ --paths '/ *'"
  }
`` ``

`XYZ` é para todas as peças que precisam ser substituídas. Agora você pode executar o `npm run deploy`, que irá implantar e invalidar

Felicidades!
