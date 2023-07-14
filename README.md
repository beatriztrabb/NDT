# NDT
#Códigos utilizados no artigo "Uma modelagem mais realista do Tempo de Descoberta de Vizinho para mecanismos de duty cycle assíncrono baseados em escalonamento"

import sys
from random import random
import math
import argparse

def progressbar(it, prefix="", size=60, out=sys.stdout): # Python3.3+
    count = len(it)
    def show(j):
        x = int(size*j/count)
        print("{}[{}{}] {}/{}".format(prefix, "#"*x, "."*(size-x), j, count), 
                end='\r', file=out, flush=True)
    show(0)
    for i, item in enumerate(it):
        yield item
        show(i+1)
    print("\n", flush=True, file=out)

def parseArgs():

	parser = argparse.ArgumentParser(description='Calcula estatisticas de NDT para escalonamentos potencialmente assimetricos com base em simulacoes numericas.')
	parser.add_argument('schAString', metavar='schA', nargs=1,
						help='Escalonamento do primeiro sensor no formato va,sa1[,sa2[,sa2...]. va eh o tamanho do ciclo, sa1,sa2,... sao os indices dos slots ativos')
	parser.add_argument('schBString', metavar='schB', nargs=1,
						help='Escalonamento do segundo sensor no formato vb,sb1[,sb2[,sb2...]. vb eh o tamanho do ciclo, sb1,sb2,... sao os indices dos slots ativos')
	parser.add_argument('p', metavar='p', type=float, nargs=1,
						help='probabilidade de sucesso do enlace')
	parser.add_argument('reps', metavar='reps', type=int, nargs=1,
						help='numero de repeticoes da simulacao')
	parser.add_argument('fatias', metavar='fatias', type=int, nargs='?', default=1,
						help='numero de fatias em que cada slot eh subdivido')
	parser.add_argument('--amostras', '-a', metavar='arquivo', default='',
						help='nome de arquivo a ser gerado com as amostras de NDT obtidas. Se \'-\' for especificado, amostras sao impressas na saida padrao. Se nao for especificado, amostras nao sao impressas.')

	args = parser.parse_args()

	schA = [int(i) for i in args.schAString[0].split(",")]
	schB = [int(i) for i in args.schBString[0].split(",")]
	p = args.p[0]
	reps = args.reps[0]
	fatias = args.fatias
	samplesFile = args.amostras
	schA_ativos = [i*fatias for i in schA[1:]]
	schB_ativos = [i*fatias for i in schB[1:]]

	va = schA[0] * fatias
	vb = schB[0] * fatias
	schA = [i*fatias + j for i in schA[1:] for j in range(fatias)]
	schB = [i*fatias + j for i in schB[1:] for j in range(fatias)]

	params = {

		"p": p,
		"va": va,
		"vb": vb,
		"schA": schA,
		"schB": schB,
		"reps": reps,
		"fatias" : fatias,
		"schA_ativos": schA_ativos,
		"schB_ativos": schB_ativos,
		"samplesFile" : samplesFile
	}

	return params

# Dado o slot atual, retorna o indice 
# do proximo slot ativo em sch
def nextActiveSlot(slot, sch):

	for i in range(len(sch)):

		if slot <= sch[i]:
			return i 
	
	return 0

# Retorna a diferenca entre dois slots, considerando
# o tamanho do ciclo do sensor
def timeUntilSlot(currentSlot, targetSlot, v):

	diff = targetSlot - currentSlot
	if diff < 0:
		diff = v + diff
	return diff

def simNDT(va, schA, vb, schB, p, schA_ativos, schB_ativos):

	# Contar o tempo de simulacao em t
	t = 0

	# Manter variavel com o tempo desde a ultima oportunidade de encontro.
	# Sera usado para detectar falta de fechamento para rotacao
	timeSinceLastOpportunity = 0

	# Para o mesmo proposito, calcular o MMC entre os comprimentos dos
	# escalonamentos. Este valor representa o tempo maximo ate que uma
	# oportunidade de encontro ocorra (assumindo fechamento para rotacao).
	maxIntervalUntilOpportunity = math.lcm(va, vb)

	# Sortear slots iniciais aleatorios para cada sensor
	slotA = math.floor(random() * va)
	slotB = math.floor(random() * vb)

	# Descobrir qual o proximo slot ativo para cada sensor
	nextActiveA = nextActiveSlot(slotA, schA)
	nextActiveB = nextActiveSlot(slotB, schB)
	#print (schA)
	#print(schA_ativos)

	# Repetir ate o encontro
	while True:

		# Descobrir quanto tempo falta ate o proximo slot ativo de 
		# um dos dois nos
		timeUntilNextActive = min(timeUntilSlot(slotA, schA[nextActiveA], va), timeUntilSlot(slotB, schB[nextActiveB], vb))

		# Avancar o tempo de simulacao
		t = t + timeUntilNextActive
		slotA = (slotA + timeUntilNextActive) % va
		slotB = (slotB + timeUntilNextActive) % vb
		
		# Para cada sensor, verificar se o proximo slot é ligado
		if slotA == schA[nextActiveA]:

			# Sim.
			activeSlotA = True

			# Avancar indice do proximo slot ativo.
			nextActiveA = (nextActiveA + 1) % len(schA)
		else:
			activeSlotA = False

		if slotB == schB[nextActiveB]:

			# Sim.
			activeSlotB = True

			# Avancar indice do proximo slot ativo.
			nextActiveB = (nextActiveB + 1) % len(schB)
		else:
			activeSlotB = False
		
			
			# Verificar se houve uma oportunidade de encontro (i.e.,
			# ambos os nos com radio ligado)
		if activeSlotA == True and activeSlotB == True:
			if schA[nextActiveA] in schA_ativos or schB[nextActiveB] in schB_ativos:
                # Sim. Sortear valor aleatorio para determinar se 
				# comunicacao foi bem sucedida.
				if random() < p:

					# Sim. Houve comunicacao. Fim da simulacao.
					return t

				# Atualizar o tempo desde a ultima oportunidade
				timeSinceLastOpportunity = 0

		else:

			# Atualizar o tempo desde a ultima oportunidade
			timeSinceLastOpportunity = timeSinceLastOpportunity + timeUntilNextActive

			# Verificar se o tempo eh maior que o maximo teorico
			if timeSinceLastOpportunity >= maxIntervalUntilOpportunity:
				print("Escalonamentos nao tem fechamento para rotacao: falha para offset {}".format(slotA - slotB))
				sys.exit(1)

params = parseArgs()
samples = [0] * params["reps"]

print("")
for i in progressbar(range(params["reps"]), "Processando: ", 40):
	samples[i] = simNDT(params["va"], params["schA"], params["vb"], params["schB"], params["p"], params["schA_ativos"], params["schB_ativos"])

print("Estatisticas:")
print("\t- NDT minimo: {}".format(min(samples) / float(params["fatias"])))
print("\t- NDT maximo: {}".format(max(samples) / float(params["fatias"])))
print("\t- NDT medio: {}".format((sum(samples) / float(params["reps"]) / float(params["fatias"]))))

print("")

if params["samplesFile"] != "":
	if params["samplesFile"] == "-":
		print("#############")
		print("Amostras:")
	else:
		f = open(params["samplesFile"], "w")
		sys.stdout = f

	for i in samples:
		print(i)
