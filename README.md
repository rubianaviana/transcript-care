# transcript-care
Script utilizado para a transcrição de um áudio longo, com mais de 240 minutos.

Script used to transcribe a long audio sample, with more than 240 minutes.

# Transcrição do Áudio  

## Objetivo  
Explorar informações fornecidas na amostra de áudio para oferecer um resumo geral de análises e insights estratégicos através da sua transcrição para formato de texto. A base de dados foi um vídeo com 4h de duração, que foi dividido em partes de até 1h para otimizar o tempo de transcrição. O script também reuniu as transcrições em um único documento de texto, e as análises foram realizadas sobre ele, reduzindo o tempo de trabalho.  

## Ferramentas  
- Python (-U openai-whisper, ffmpeg)  
- Jupyter Notebook  

## Resultados  
Transcrição 

<img width="1366" height="768" alt="image" src="https://github.com/user-attachments/assets/890ef250-a3f4-4b12-9da5-8f4c4a034302" />

  

## Como Executar  
```bash
!pip install -U openai-whisper
!sudo apt update && sudo apt install ffmpeg -y

# Configurações
input_file = "videoplayback.mp4" #Aqui vai o seu arquivo de áudio
output_dir = "output_whisper"
model_name = "medium"  # Mude para "large-v3" se precisar de mais precisão, nesse formato o script levou cerca 56min para concluir
segment_duration = 3600  # 1 hora em segundos

# Verifica se o arquivo existe
if not os.path.exists(input_file):
    print(f"Erro: Arquivo '{input_file}' não encontrado!")
    exit()

# Cria diretório de saída
os.makedirs(output_dir, exist_ok=True)

# 1. Converter para WAV e dividir em partes de 1 hora
print("\nConvertendo e dividindo o áudio...")
try:
    # Primeiro converte para WAV
    temp_wav = os.path.join(output_dir, "temp_audio.wav")
    subprocess.run([
        "ffmpeg",
        "-i", input_file,
        "-vn", "-acodec", "pcm_s16le",
        "-ar", "16000", "-ac", "1",
        temp_wav
    ], check=True)

    # Depois divide em partes
    subprocess.run([
        "ffmpeg",
        "-i", temp_wav,
        "-f", "segment",
        "-segment_time", str(segment_duration),
        "-c", "copy",
        os.path.join(output_dir, "parte_%03d.wav")
    ], check=True)

    # Remove o arquivo temporário
    os.remove(temp_wav)

    print("Áudio dividido com sucesso em partes de 1 hora!")
except subprocess.CalledProcessError as e:
    print(f"Erro ao processar áudio: {e}")
    exit()

# 2. Transcrever cada parte
partes = sorted([f for f in os.listdir(output_dir) if f.startswith("parte_") and f.endswith(".wav")])
transcricao_completa = ""

for i, parte in enumerate(partes, 1):
    parte_path = os.path.join(output_dir, parte)
    output_name = os.path.splitext(parte)[0]  # Remove a extensão .wav
    output_txt = os.path.join(output_dir, f"{output_name}.txt")  # Usa o mesmo nome da parte

    print(f"\nTranscrevendo parte {i}/{len(partes)}: {parte}")
    start_time = datetime.now()

    try:
        # Executa o Whisper com output_file específico
        result = subprocess.run([
            "whisper",
            parte_path,
            "--language", "pt",
            "--model", model_name,
            "--output_dir", output_dir,
            "--output_format", "txt",
            "--task", "transcribe",
            "--verbose", "True"
        ], capture_output=True, text=True)

        # Verifica se o arquivo foi criado
        if not os.path.exists(output_txt):
            print(f"Aviso: O arquivo {output_txt} não foi criado!")
            print("Saída do Whisper:")
            print(result.stdout)
            print("Erros:")
            print(result.stderr)
            continue

        # Adiciona à transcrição completa
        with open(output_txt, 'r', encoding='utf-8') as f:
            conteudo = f.read()
            transcricao_completa += f"=== Parte {i} ===\n{conteudo}\n\n"

        print(f"Parte {i} concluída em {datetime.now() - start_time}")

    except Exception as e:
        print(f"Erro ao transcrever {parte}: {str(e)}")
        continue

# 3. Salvar transcrição completa
if transcricao_completa:
    final_output = os.path.join(output_dir, "transcricao_completa.txt")
    with open(final_output, 'w', encoding='utf-8') as f:
        f.write(transcricao_completa)

    print("\n" + "="*50)
    print(f"Transcrição completa salva em: {final_output}")
    print(f"Tamanho total do texto: {len(transcricao_completa.split())} palavras")
    print("Prévia da transcrição:")
    print(transcricao_completa[:500] + "...")

    # Faz download no Colab
    if 'google.colab' in str(get_ipython()):
        from google.colab import files
        files.download(final_output)
else:
    print("\nErro: Nenhuma transcrição foi gerada.")
