# Aplicação de Boletos Omie

- Imports: Dependencias do projeto.
- moment é para formatar datas, porque usa o calendário html5 no frontend.
- _async fluxo()_ é o método principal, que é chamado pelo servidor.

```typescript
import {includes, JSON_HEADER, NOT_FOUND, readRequestBody} from './Utils';
import moment from 'moment-timezone';
import 'moment/locale/pt-br';

export class Apps {

    private readonly request: Request;
    private readonly env: Env;
    private readonly ctx: ExecutionContext;

    constructor(request: Request, env: Env, ctx: ExecutionContext) {
        this.request = request;
        this.env = env;
        this.ctx = ctx;
    }

    async fluxo() {
        try {
            let data = await readRequestBody(this.request);

            if (includes(this.request.url, '/listaboletos')) {
                let resposta = await listaDeBoleto(this.env,data);

                return new Response(
                    JSON.stringify(resposta),
                    {
                        headers: JSON_HEADER,
                        status: 200,
                    },
                );

            } else if (includes(this.request.url, '/urlboleto')) {
                return await urlBoleto(this.env,this.request.url.split('/').pop());
            }

        } catch (e) {
            console.error('Omie App', e, e.stack);
            return new Response('Erro', {status: 500});
        }
        return NOT_FOUND();
    }
}
```

# Listagem dos boletos

```typescript

export async function listaDeBoleto(env : Env,data:any) {
    let resposta = [];

    try {
        let retornoOmie = await fetch(
            'https://app.omie.com.br/api/v1/financas/pesquisartitulos/',
            {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                    'Accept': 'application/json',
                },
                body: JSON.stringify({
                    'call': 'PesquisarLancamentos',
                    'app_key': env.OMIE_APP_KEY,
                    'app_secret': env.OMIE_APP_SECRET,
                    'param': [
                        {
                            'dDtIncDe': moment(data['datade'], 'YYYY-MM-DD').format('DD/MM/YYYY'),
                            'dDtIncAte': moment(data['dataate'], 'YYYY-MM-DD').format('DD/MM/YYYY'),
                            'cNatureza': 'R',
                            'cTipo': 'BOL',
                            //'cStatus': 'AVENCER',
                            'nRegPorPagina': 200,
                        },
                    ],
                }),
            },
        );
        resposta = (await retornoOmie.json());

        if (!resposta['titulosEncontrados']) {
            return [];
        }

        let boletos = [];
        for (let i = 0; i < resposta['titulosEncontrados'].length; i++) {
            boletos.push(await dtoBoleto(resposta['titulosEncontrados'][i]['cabecTitulo']));
        }
        resposta = boletos;

        resposta = resposta.sort(sorter);

    } catch (e) {
        console.error('Erro ao listar contas a receber', e, e.stack);
    }
    return resposta;
}
```

# Ordenação dos boletos

```typescript

export function stringToSort(entity: any): string {
    return entity.cNumTitulo + '-' + entity.cNumParcela;
}

export function sorter(a: any, b: any): number {
    return stringToSort(a).localeCompare(stringToSort(b));
}

export function dtoBoleto(titulo: any) {
    return {
        'cCPFCNPJCliente': titulo.cCPFCNPJCliente,
        'nCodTitulo': titulo.nCodTitulo,
        'cNumTitulo': titulo.cNumTitulo ? titulo.cNumTitulo :
            titulo.cNumOS ? titulo.cNumOS :
                titulo.cNumDocFiscal ? titulo.cNumDocFiscal :
                    titulo.nCodTitulo
        ,
        'cNumParcela': titulo.cNumParcela,
        'nValorTitulo': titulo.nValorTitulo,
        'cCodigoBarras': titulo.cCodigoBarras,
    };
}

```

# Responder com o binario do Boleto em PDF

-- usar cache para PDF - https://developers.cloudflare.com/workers/examples/cache-api/

```typescript
export async function urlBoleto(env: Env, codigo: string) {
    return new Response(
        await (await fetch(
            (await (await fetch(
                'https://app.omie.com.br/api/v1/financas/pesquisartitulos/',
                {
                    method: 'POST',
                    headers: {
                        'Content-Type': 'application/json',
                        'Accept': 'application/json',
                    },
                    body: JSON.stringify({
                        'call': 'ObterURLBoleto',
                        'app_key': env.OMIE_APP_KEY,
                        'app_secret': env.OMIE_APP_SECRET,
                        'param': [
                            {
                                'nCodTitulo': `${codigo}`,
                            },
                        ],
                    }),
                },
            )).json())['cLinkBoleto']
        )).blob(),
        {
            status: 200
        }
    )
}
```
