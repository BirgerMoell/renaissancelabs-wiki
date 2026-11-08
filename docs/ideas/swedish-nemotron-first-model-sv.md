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
2. **Preferensträning med [DPO](https://arxiv.org/abs/2305.18290)** på svenska värderingar, så att modellens svar speglar de normer vi vill att den ska hålla. Utkastet [Svensk Konstitution](https://github.com/BirgerMoell/svensk-konstitution) (en öppen, ogranskad v0.1-text om värderingar och omdöme för AI-agenter som arbetar på svenska, CC BY 4.0) kan hjälpa på två sätt: som underlag för att generera preferenspar, där en lärarmodell svarar, kritiserar och reviderar mot texten och studentmodellen sedan tränas utan texten i prompten, och som rubrik för att avgöra vilket svar som är bäst.
3. **Förstärkningsinlärning för bra svenska.** För varje uppgift genererar modellen flera svar (cirka 8), en rubrikbaserad domarmodell rangordnar dem och [GRPO](https://arxiv.org/abs/2402.03300) flyttar modellen mot de bästa. Att använda en AI-domare i stället för mänskliga bedömare kallas [RLAIF](https://arxiv.org/abs/2309.00267). Rubriken kan bygga på [Skrivbench](../projects/skrivbench.md). [Svenska RL](https://github.com/BirgerMoell/svenska-rl) bidrar med verifierbara svenska miljöer (bland annat fakturabelopp, betalningsmatchning, mötesbokning, maskning av personuppgifter och verktygsuppgifter). De bedöms genom att resultatet kontrolleras och inte av en domare, så de kompletterar rubrikdomaren. Åtta miljöer finns i dag; de 20 000 som nämns i repot är ett planeringsmål.

## Öppna frågor

**Ska modellen resonera på svenska?** Ett tidigare experiment fick en Nemotron-modell att resonera på svenska. Att resonera på ett annat språk än engelska ger oftast sämre prestanda överlag, men svenskt resonemang är ändå värdefullt att ha. Ett alternativ är att träna två varianter, en med svenskt resonemang och en utan, och jämföra dem. Vår nuvarande hållning är att svenskt resonemang bör vara ett krav och att vi i så fall får lösa prestandaförlusten, vilket kan bli svårt. Vi vill gärna höra andra åsikter.

**Vilken data ska vi använda?**
- Översatta instruktions- och andra dataset
- Öppen svensk text, bland annat material från regering och riksdag
- Syntetisk data genererad med [Kimi K3](https://huggingface.co/moonshotai/Kimi-K3), Codex och Claude

**Hur ska vi utvärdera?** Utvärderingen måste spegla våra mål. Ett första steg kan vara enkelt: bra svenska, och ingen försämring på övriga förmågor.

## Datakällor

Licenserna är de som anges på respektive Hugging Face-sida. Där sidan anger "other" eller ingen licens alls bör man kontrollera innan man använder den, och syntetisk data kan också följa den modells villkor som genererade den. Översatta dataset tenderar att innehålla anglicismer, vilket är just vad Skrivbench mäter, så vi vill blanda dem med originalsvensk text.

**Förträning och fortsatt förträning**

| Källa | Vad det är | Licens |
|---|---|---|
| [FineWeb-2](https://huggingface.co/datasets/HuggingFaceFW/fineweb-2) (`swe_Latn`) | Svensk webbtext | ODC-By |
| [FinePDFs](https://huggingface.co/datasets/HuggingFaceFW/finepdfs) och [FinePDFs-Edu](https://huggingface.co/datasets/HuggingFaceFW/finepdfs-edu) (`swe_Latn`) | Långa dokument extraherade ur PDF-filer, väl lämpade för lång kontext | ODC-By |
| [FineWiki](https://huggingface.co/datasets/HuggingFaceFW/finewiki) (`sv`) | Rensad svensk Wikipedia | CC BY-SA 4.0, GFDL |
| [HPLT 2.0 cleaned](https://huggingface.co/datasets/HPLT/HPLT2.0_cleaned) (`swe_Latn`) | Webbtext | CC0 |
| [Swedish Dynaword](https://huggingface.co/datasets/danish-foundation-models/swedish-dynaword) | Kurerad öppen svensk text, bland annat statliga utredningar (SOU), historiska lagar och tidskrifter | CC0 |
| [RixVox v2](https://huggingface.co/datasets/KBLab/rixvox-v2) | Riksdagstal med transkriptioner (KBLab) | ODC-By |

För lång kontext omvandlar [openeuro-longctx-datamix](https://github.com/BirgerMoell/openeuro-longctx-datamix) FinePDFs-Edu, svenska inkluderad, till en viktad datamix för lång kontext. [SWEb](https://arxiv.org/abs/2410.04456) är en stor skandinavisk webbkorpus, men vi kunde inte bekräfta någon offentlig datasetsida, så kontrollera åtkomsten först.

**Övervakad finjustering**

| Källa | Vad det är | Licens |
|---|---|---|
| [Dolci-Instruct-SFT-translated](https://huggingface.co/datasets/openeurollm/Dolci-Instruct-SFT-translated) (`sv`) | Instruktionsdata översatt av OpenEuroLLM | Apache-2.0 |
| [Dolci-Think-SFT-translated](https://huggingface.co/datasets/openeurollm/Dolci-Think-SFT-translated) (`sv`) | Resonemangsdata, översatt | Apache-2.0 |
| [EuroMultiSynth-Instruct](https://huggingface.co/datasets/openeurollm/EuroMultiSynth-Instruct) (`sv`) | Flerspråkig syntetisk instruktionsdata | Apache-2.0 |
| [smol-smoltalk-swedish](https://huggingface.co/datasets/liu-nlp/smol-smoltalk-swedish) | Cirka 457 000 svenska konversationer (Linköpings universitet) | Ej angiven |
| [reasoning-traces-multilingual](https://huggingface.co/datasets/openeurollm/reasoning-traces-multilingual) | Flerspråkiga resonemangsspår, svenska inkluderad | CC BY 4.0 |
| [openeurollm-reasoning-instruct-v1](https://huggingface.co/datasets/birgermoell/openeurollm-reasoning-instruct-v1) och [openeurollm-tool-use-v1](https://huggingface.co/datasets/birgermoell/openeurollm-tool-use-v1) | Resonemangs- och verktygsdata | Other |

**Preferensdata (DPO)**

| Källa | Vad det är | Licens |
|---|---|---|
| [Dolci-Instruct-DPO-translated](https://huggingface.co/datasets/openeurollm/Dolci-Instruct-DPO-translated) | Preferenspar översatta av OpenEuroLLM | Apache-2.0 |
| [dpo-swedish-interpreted](https://huggingface.co/datasets/jonasaise/dpo-swedish-interpreted) | UltraFeedback-par anpassade till svenska med Kimi K3. De avvisade svaren är dåliga översättningar, så den lär ut naturlig formulering mer än värderingar. | MIT |
| [oellm-dpo-metadataset](https://huggingface.co/datasets/birgermoell/oellm-dpo-metadataset) | DPO-samling med svenska inkluderad. Vi har ännu inte granskat innehållet. | Apache-2.0 |

**Förstärkningsinlärning och prompter**

- [Svenska RL](https://github.com/BirgerMoell/svenska-rl): verifierbara svenska miljöer, med export till RL-träningsdata.
- [oellm-math-rlvr](https://huggingface.co/datasets/openeurollm/oellm-math-rlvr): matematikuppgifter med kontrollerbara svar (Apache-2.0).
- [swedish-personas](https://huggingface.co/datasets/birgermoell/swedish-personas): 100 000 syntetiska svenska personer dragna ur SCB:s demografi, användbara för att variera prompter (CC BY 4.0).

**Kataloger och närliggande arbete.** [OpenEuroLLM post-training data registry](https://huggingface.co/datasets/openeurollm/post-training-data-registry), katalogen [post-training-data](https://github.com/BirgerMoell/post-training-data) och [training-data-catalogue](https://github.com/OpenEuroLLM/training-data-catalogue) förtecknar vad som finns över språk. [nemotron-swedish-posttraining](https://github.com/BirgerMoell/nemotron-swedish-posttraining) innehåller tidigare svenska eftertränings-experiment för Nemotron, och [svea-eval](https://github.com/BirgerMoell/svea-eval) och [svbench-v2](https://github.com/BirgerMoell/svbench-v2) är svenska utvärderingar vi kan använda för kontrollen att inget försämras. Fler ställen att leta på: [birgermoell på Hugging Face](https://huggingface.co/birgermoell), [openeurollm på Hugging Face](https://huggingface.co/openeurollm), [BirgerMoell på GitHub](https://github.com/BirgerMoell) och [OpenEuroLLM på GitHub](https://github.com/OpenEuroLLM).

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
