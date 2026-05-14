# Api-riovagasfrom fastapi import FastAPI, HTTPException
import requests
from bs4 import BeautifulSoup

app = FastAPI()

# --- CAMADA DO ROBÔ (SCRAPER) ---
def extrair_dados_riovagas(cargo_busca: str):
    """
    Esta função é o seu 'robô'. Ela entra no site e extrai a informação bruta.
    """
    # 1. Monta a URL de busca (O RioVagas usa ?s= para pesquisas)
    url = f"https://riovagas.com.br/?s={cargo_busca}"
    
    # 2. Identificação para o site não nos bloquear (User-Agent)
    headers = {
        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36"
    }
    
    try:
        # 3. Faz o pedido (Request)
        resposta = requests.get(url, headers=headers, timeout=10)
        resposta.raise_for_status() # Se der erro 404 ou 500, ele avisa aqui
        
        # 4. Transforma o HTML em um objeto 'navegável'
        sopa = BeautifulSoup(resposta.text, 'html.parser')
        
        vagas_encontradas = []
        
        # 5. Localiza os blocos de vaga (No RioVagas geralmente são tags <article>)
        for artigo in sopa.find_all('article'):
            link_tag = artigo.find('a')
            if link_tag:
                titulo = link_tag.text.strip()
                link = link_tag['href']
                
                # Tenta pegar um resumo se existir
                resumo_tag = artigo.find('p')
                resumo = resumo_tag.text.strip() if resumo_tag else "Sem descrição disponível"
                
                vagas_encontradas.append({
                    "titulo": titulo,
                    "link": link,
                    "resumo": resumo
                })
        
        return vagas_encontradas

    except Exception as e:
        return {"erro": f"Falha na extração: {str(e)}"}

# --- CAMADA DA API (ROTAS) ---

@app.get("/api/v1/vagas")
async def listar_vagas(cargo: str):
    """
    Este é o endpoint que o usuário (ou outro sistema) vai chamar.
    """
    if not cargo:
        raise HTTPException(status_code=400, detail="Por favor, informe um cargo para busca.")
        
    resultado = extrair_dados_riovagas(cargo)
    
    # Se o robô retornou um erro
    if isinstance(resultado, dict) and "erro" in resultado:
        raise HTTPException(status_code=500, detail=resultado["erro"])
        
    return {
        "status": "sucesso",
        "total_vagas": len(resultado),
        "resultados": resultado
    }

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="127.0.0.1", port=8000)
