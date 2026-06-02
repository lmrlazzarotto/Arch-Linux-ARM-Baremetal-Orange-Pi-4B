# Arch-Linux-ARM-Baremetal-Orange-Pi-4B
Este repositório documenta o processo bruto e tático para construir, do absoluto zero, um cartão SD inicializável com Arch Linux ARM para a série Orange Pi 4 (focado no modelo 4B).
A documentação oficial para processadores Rockchip é fragmentada e não oferece uma receita pronta para esta placa específica.
Se você errar um offset do U-Boot ou usar a ferramenta de extração errada, o sistema simplesmente não vai dar boot e você vai perder seu tempo fazendo troubleshooting cego via porta serial.
O objetivo aqui é estabelecer uma base de sistema operacional limpa, rastreável e sem bloatware, ideal para subir um home server estável e focado em desempenho.
Trate cada etapa abaixo como uma migração de dados em um ambiente de produção: exija precisão, não pegue atalhos e valide seus comandos. 
Coloque um metal no fone de ouvido para não dormir durante o comando dd, identifique seu cartão SD corretamente e siga o plano.
