# Träna Nemotron 3.5 Lightning på svenska

*[English version](swedish-nemotron-first-model.md)*

**Status:** förslag, öppet för synpunkter
**Författare:** Renaissance Labs

Vi överväger [Nemotron 3.5 Lightning](https://huggingface.co/nvidia/NVIDIA-Nemotron-3.5-Lightning-30B-A3B-BF16) (30B-A3B) som första modell att träna för svenska. Den här sidan beskriver varför, hur vi tänker träna den och vilka frågor vi ännu inte har svar på.

## Varför just den här modellen

- **Arkitektur.** Den är en hybrid av [Mamba-2](https://arxiv.org/abs/2405.21060)-lager (tillståndsrumsmodeller), [mixture-of-experts](https://huggingface.co/blog/moe)-lager och några få attention-lager. Den har 30 miljarder parametrar totalt, ungefär 3 miljarder aktiva per token, och ett kontextfönster på upp till 1 miljon tokens enligt [NVIDIA:s dokumentation](https://docs.api.nvidia.com/nim/reference/nvidia-nemotron-3-5-lightning-30b-a3b). Tjänster från andra leverantörer har ofta en lägre standardgräns, till exempel 262 000, så kontrollera inställningen i det du faktiskt kör.
- **Storlek.** Den är så pass liten att vi kan träna och utvärdera den utan stor beräkningstilldelning, och den kvantiserade [NVFP4](https://huggingface.co/nvidia/NVIDIA-Nemotron-3.5-Lightning-30B-A3B-NVFP4)-versionen gör utvärdering lokalt på en laptop realistisk. Samtidigt är den stor nog att vara användbar, och billig nog att en misslyckad körning inte gör ont.
- **Dogfooding.** När modellen är tränad förbinder vi oss att använda den för vårt eget svenska skrivande. Vi tvingas då upptäcka och rätta de brister vi annars skulle undvika.

## Träningsplan

1. **Övervakad finjustering ([SFT](https://huggingface.co/docs/trl/sft_trainer))** på svensk text, tillsammans med data för resonemang och verktygsanvändning på både svenska och engelska.
2. **Preferensträning med [DPO](https://arxiv.org/abs/2305.18290)** på svenska värderingar, så att modellens svar speglar de normer vi vill att den ska hålla.
3. **Förstärkningsinlärning för bra svenska.** För varje uppgift genererar modellen flera svar (cirka 8), en rubrikbaserad domarmodell rangordnar dem och [GRPO](https://arxiv.org/abs/2402.03300) flyttar modellen mot de bästa. Att använda en AI-domare i stället för mänskliga bedömare kallas [RLAIF](https://arxiv.org/abs/2309.00267). Rubriken kan bygga på [Skrivbench](../projects/skrivbench.md).

## Öppna frågor

**Ska modellen resonera på svenska?** Ett tidigare experiment fick en Nemotron-modell att resonera på svenska. Att resonera på ett annat språk än engelska ger oftast sämre prestanda överlag, men svenskt resonemang är ändå värdefullt att ha. Ett alternativ är att träna två varianter, en med svenskt resonemang och en utan, och jämföra dem. Vår nuvarande hållning är att svenskt resonemang bör vara ett krav och att vi i så fall får lösa prestandaförlusten, vilket kan bli svårt. Vi vill gärna höra andra åsikter.

**Vilken data ska vi använda?**
- Översatta instruktions- och andra dataset
- Öppen svensk text, bland annat material från regering och riksdag
- Syntetisk data genererad med [Kimi K3](https://huggingface.co/moonshotai/Kimi-K3), Codex och Claude

**Hur ska vi utvärdera?** Utvärderingen måste spegla våra mål. Ett första steg kan vara enkelt: bra svenska, och ingen försämring på övriga förmågor.

## Hur vi kör experimenten

- **Börja smått.** Börja med en [LoRA](https://arxiv.org/abs/2106.09685)-adapter, eller en full träningskörning om kostnaden visar sig acceptabel.
- **Budgetera före varje körning.** Fråga vad körningen kostar i GPU-timmar (en kodagent kan uppskatta det), vilken förmåga den tränar och vad vi förväntar oss lära oss. Små experiment kommer först.
- **Planera sedan den stora körningen.** När de små körningarna har visat vad som fungerar strukturerar vi datan, och en person tar ansvar för själva träningen.
- **Kör på klustret.** Att tveka att starta körningar är det dyrare misstaget. Håll koll på budgeten och antalet GPU:er, och sätt i övrigt igång. Projektet [OpenEuroLLM](https://openeurollm.eu), som tränar öppna europeiska språkmodeller, är en bra referens för hur stora flerspråkiga satsningar organiserar arbetet.

## Länkar

- Modell: [Hugging Face (BF16)](https://huggingface.co/nvidia/NVIDIA-Nemotron-3.5-Lightning-30B-A3B-BF16), [NVIDIA NIM-dokumentation](https://docs.api.nvidia.com/nim/reference/nvidia-nemotron-3-5-lightning-30b-a3b)
- Mamba: [Mamba](https://arxiv.org/abs/2312.00752) och [Mamba-2](https://arxiv.org/abs/2405.21060)
- Metoder: [LoRA](https://arxiv.org/abs/2106.09685), [DPO](https://arxiv.org/abs/2305.18290), [GRPO](https://arxiv.org/abs/2402.03300), [RLAIF](https://arxiv.org/abs/2309.00267)
- Datagenerator: [Kimi K3](https://huggingface.co/moonshotai/Kimi-K3)
- Projektreferens: [OpenEuroLLM](https://openeurollm.eu)
