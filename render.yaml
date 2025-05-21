const express = require('express');
const admin = require('firebase-admin');
const { getDistance } = require('geolib');
const cors = require('cors');

// Inicializa Firebase Admin (usa credenciais padrão do ambiente)
admin.initializeApp({
  credential: admin.credential.applicationDefault(),
});
const db = admin.firestore();


const app = express();
app.use(cors({
  origin: '*', // ou especifique o domínio em produção
  methods: ['GET', 'POST', 'OPTIONS'],
  allowedHeaders: ['Content-Type'],
}));
app.use(express.json());

// Função que espera resposta do motorista (aceitar ou recusar)
function aguardarRespostaMotorista(corridaRef, motoristaId, segundos = 15) {
  return new Promise((resolve) => {
    const unsubscribe = corridaRef.onSnapshot((doc) => {
      const data = doc.data();
      if (data.status === 'aceita' && data.motorista_id_atual === motoristaId) {
        unsubscribe();
        resolve(true);
      }
      if (data.status === 'recusada' && data.motorista_id_atual === motoristaId) {
        unsubscribe();
        resolve(false);
      }
    });

    // Timeout para não ficar esperando indefinidamente
    setTimeout(() => {
      unsubscribe();
      resolve(false);
    }, segundos * 1000);
  });
}

// Função principal que busca motorista mais próximo
async function buscarMotoristaProximo(corridaId, origem) {
  const corridaRef = db.collection('corridas').doc(corridaId);

  // Buscar motoristas online e disponíveis
  const snapshot = await db.collection('drivers')
    .where('isOnline', '==', true)
    .where('analise', '==', false)
    .get();

  if (snapshot.empty) {
    console.log('Nenhum motorista disponível');
    await corridaRef.update({ status: 'sem_motoristas' });
    return;
  }

  // Calcular distâncias
  const motoristasComDistancia = snapshot.docs.map(doc => {
    const data = doc.data();
    const distancia = getDistance(origem, data.location); // em metros
    return {
      id: doc.id,
      ...data,
      distancia
    };
  });

  // Ordenar por distância
  motoristasComDistancia.sort((a, b) => a.distancia - b.distancia);

  const tentados = [];

  for (const motorista of motoristasComDistancia) {
    tentados.push(motorista.id);
    console.log(`Enviando corrida para motorista ${motorista.id}...`);

    // Marcar o motorista como em análise
    await db.collection('drivers').doc(motorista.id).update({ analise: true });

    // Atualizar corrida com motorista atual
    await corridaRef.update({
      driverRef: motorista.id,
      motoristas_tentados: tentados,
      status: 'aguardando_motorista'
    });

    const aceitou = await aguardarRespostaMotorista(corridaRef, motorista.id, 15);

    if (aceitou) {
      console.log(`Motorista ${motorista.id} aceitou a corrida.`);
      return;
    } else {
      console.log(`Motorista ${motorista.id} recusou ou não respondeu.`);
      await db.collection('drivers').doc(motorista.id).update({ analise: false });
    }
  }

  // Nenhum motorista aceitou
  await corridaRef.update({ status: 'sem_motoristas' });
  console.log('Todos motoristas recusaram ou não responderam.');
}

// Rota HTTP para disparar a busca (chame essa rota via POST)
// Exemplo JSON do body:
// {
//   "corridaId": "id_da_corrida",
//   "origem": { "latitude": -23.5, "longitude": -46.6 }
// }
app.post('/buscar-motorista-proximo', async (req, res) => {
  const { corridaId, origem } = req.body;
  if (!corridaId || !origem || !origem.latitude || !origem.longitude) {
    return res.status(400).json({ error: 'Parâmetros inválidos' });
  }

  try {
    await buscarMotoristaProximo(corridaId, origem);
    res.status(200).json({ message: 'Processo iniciado com sucesso' });
  } catch (error) {
    console.error('Erro na busca do motorista:', error);
    res.status(500).json({ error: 'Erro interno no servidor' });
  }
});

// Start do servidor
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Servidor rodando na porta ${PORT}`);
});
