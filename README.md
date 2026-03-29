import requests
import csv
import time

# --- CONFIGURACIÓN ---
TOKEN = '322caf8ec2314a1f9bf47be3be43f69b' 
TOURNAMENT_SLUG = 'stranger-spins-34-corte-circular-edition'
URL_API = 'https://api.start.gg/gql/alpha'

headers = {'Authorization': f'Bearer {TOKEN}'}

def limpiar_nick(nombre):
    if '|' in nombre:
        return nombre.split('|')[-1].strip()
    return nombre.strip()

# Consulta para encontrar el evento con más participantes
query_buscar_principal = """
query GetMainEvent($slug: String) {
  tournament(slug: $slug) {
    events { id name numEntrants }
  }
}
"""

# Consulta para posiciones (Standings)
query_standings = """
query EventStandings($eventId: ID!, $page: Int) {
  event(id: $eventId) {
    standings(query: { page: $page, perPage: 50 }) {
      nodes {
        placement
        entrant { id name }
      }
    }
  }
}
"""

# Consulta para sets y scores
query_sets = """
query EventSets($eventId: ID!, $page: Int) {
  event(id: $eventId) {
    sets(page: $page, perPage: 50) {
      nodes {
        slots {
          entrant { id }
          standing { stats { score { value } } }
        }
      }
    }
  }
}
"""

def procesar_torneo():
    print(f"--- PROCESANDO TORNEO: {TOURNAMENT_SLUG} ---")
    
    # 1. SELECCIONAR EVENTO PRINCIPAL
    resp = requests.post(URL_API, json={'query': query_buscar_principal, 'variables': {'slug': TOURNAMENT_SLUG}}, headers=headers)
    eventos = resp.json()['data']['tournament']['events']
    evento_principal = max(eventos, key=lambda x: x['numEntrants'] if x['numEntrants'] else 0)
    ev_id = evento_principal['id']
    ev_name = evento_principal['name']
    print(f"Evento detectado: {ev_name} ({evento_principal['numEntrants']} participantes)")

    # 2. DESCARGAR POSICIONES Y MAPEAR JUGADORES
    id_to_clean_name = {}
    posiciones_finales = []
    page = 1
    while True:
        r = requests.post(URL_API, json={'query': query_standings, 'variables': {'eventId': ev_id, 'page': page}}, headers=headers)
        nodes = r.json()['data']['event']['standings']['nodes']
        if not nodes: break
        for n in nodes:
            if n['entrant']:
                nick = limpiar_nick(n['entrant']['name'])
                id_to_clean_name[n['entrant']['id']] = nick
                posiciones_finales.append([nick, n['placement']])
        page += 1

    # 3. DESCARGAR SETS
    all_sets = []
    page = 1
    print(f"Descargando historial de enfrentamientos...")
    while True:
        r = requests.post(URL_API, json={'query': query_sets, 'variables': {'eventId': ev_id, 'page': page}}, headers=headers)
        nodes = r.json()['data']['event']['sets']['nodes']
        if not nodes: break
        all_sets.extend(nodes)
        page += 1
        time.sleep(0.1)

    # 4. CONSTRUIR MATRIZ DE HISTORIAL
    nombres_ordenados = sorted(list(set(id_to_clean_name.values())))
    name_to_idx = {n: i for i, n in enumerate(nombres_ordenados)}
    n = len(nombres_ordenados)
    matriz_historial = [[""]*n for _ in range(n)]

    for s in all_sets:
        slots = s.get('slots', [])
        if len(slots) < 2 or not slots[0]['entrant'] or not slots[1]['entrant']:
            continue
            
        id1, id2 = slots[0]['entrant']['id'], slots[1]['entrant']['id']
        if id1 in id_to_clean_name and id2 in id_to_clean_name:
            nom1, nom2 = id_to_clean_name[id1], id_to_clean_name[id2]
            idx1, idx2 = name_to_idx[nom1], name_to_idx[nom2]
            
            s1 = slots[0]['standing']['stats']['score']['value']
            s2 = slots[1]['standing']['stats']['score']['value']
            
            val1 = str(max(0, int(s1))) if s1 is not None else "0"
            val2 = str(max(0, int(s2))) if s2 is not None else "0"
            
            # Guardamos historial (si se enfrentan 2 veces aparecerá "2, 0")
            matriz_historial[idx1][idx2] += (", " if matriz_historial[idx1][idx2] else "") + val1
            matriz_historial[idx2][idx1] += (", " if matriz_historial[idx2][idx1] else "") + val2

    # 5. GUARDAR ARCHIVOS
    f_pos = f"{TOURNAMENT_SLUG}_posiciones.csv"
    f_mat = f"{TOURNAMENT_SLUG}_matriz_de_games.csv"

    # Guardar Posiciones
    with open(f_pos, 'w', newline='', encoding='utf-8-sig') as f:
        f.write("sep=;\n")
        writer = csv.writer(f, delimiter=';')
        writer.writerow(['Jugador', 'Posicion'])
        writer.writerows(posiciones_finales)

    # Guardar Matriz
    with open(f_mat, 'w', newline='', encoding='utf-8-sig') as f:
        f.write("sep=;\n")
        writer = csv.writer(f, delimiter=';')
        writer.writerow(['Fila gana a Columna (Historial)'] + nombres_ordenados)
        for i, fila in enumerate(matriz_historial):
            writer.writerow([nombres_ordenados[i]] + fila)

    print("-" * 30)
    print(f"¡FINALIZADO!")
    print(f"1. {f_pos}")
    print(f"2. {f_mat}")

if __name__ == "__main__":
    procesar_torneo()
