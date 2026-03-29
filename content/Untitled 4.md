```
https://mielingreso.unlam.edu.ar/contenido/event/descargarElemento/comision/30161/a/INGRESO_2021008%5C_RESOLUCION--EXAMEN-GEOMETRIA-14-hs-OCTUBRE-2025--TEMAS-1-Y-2.pdf/id/7943
```
```
https://mielingreso.unlam.edu.ar/contenido/event/descargarElemento/comision/30257/a/INGRESO_2021014%5C_RESOLUCION--EXAMEN-SEMINARIO-Y-PRODUCCION-DE-TEXTO-14-hs-DICIEMBRE-2024--TEMAS-1-Y-2.pdf/id/7358
```
```
https://mielingreso.unlam.edu.ar/contenido/event/descargarElemento/comision/30257/a/INGRESO_2021014%5C_RESOLUCION--EXAMEN-SEMINARIO-Y-PRODUCCION-DE-TEXTO-14-hs-DICIEMBRE-2024--TEMAS-1-Y-2.pdf/id/7357
```
```
https://mielingreso.unlam.edu.ar/contenido/event/descargarElemento/comision/30257/a/INGRESO_2021014%5C_RESOLUCION--EXAMEN-SEMINARIO-Y-PRODUCCION-DE-TEXTO-10-hs-DICIEMBRE-2024--TEMAS-1-Y-2.pdf/id/7353-60
```
```
https://mielingreso.unlam.edu.ar/contenido/event/descargarElemento/comision/30257/a/INGRESO_2021014%5C_RESOLUCION--EXAMEN-SEMINARIO-Y-PRODUCCION-DE-TEXTO-14-hs-DICIEMBRE-2024--TEMAS-1-Y-2.pdf/id/7364-70
```
```
sudo docker run -v $(pwd)/wordlist:/wordlist/ -it ghcr.io/xmendez/wfuzz wfuzz 
```

https://mielingreso.unlam.edu.ar/contenido/event/descargarElemento/comision/30161/a/INGRESO_2021008%5C_RESOLUCION--EXAMEN-GEOMETRIA-14-hs-OCTUBRE-2025--TEMAS-1-Y-2.pdf/id/7943
```
sudo docker run -v $(pwd)/wordlist:/wordlist/ -it ghcr.io/xmendez/wfuzz wfuzz \
  -z range,50-70 \
  -b "_ga_9SLXM9GQ7R=GS2.1.s1764981621$o13$g1$t1764981651$j30$l0$h0; _ga=GA1.1.992468134.1759064775; PHPSESSID=o58teels5igespn6nmj6gsqpe5; SESSID=nodo8" \
  -t 1 \
  -s 3 \
  mielingreso.unlam.edu.ar/contenido/event/descargarElemento/comision/30257/a/INGRESO_2021014%5C_RESOLUCION--EXAMEN-SEMINARIO-DE-COMPRENSION-Y-PRODUCCION-DE-TEXTO-10-hs-DICIEMBRE-2025--TEMAS-1-Y-2.pdf/id/73FUZZ

```
```
sudo docker run -v $(pwd)/wordlist:/wordlist/ -it ghcr.io/xmendez/wfuzz wfuzz \
  -z range,50-70 \
  -u "https://mielingreso.unlam.edu.ar/contenido/event/descargarElemento/comision/30257/a/INGRESO_2021014%5C_RESOLUCION--EXAMEN-SEMINARIO-10-hs-DICIEMBRE-2024--TEMAS-1-Y-2.pdf/id/73FUZZ" \
  -b "_ga_9SLXM9GQ7R=GS2.1.s1764978507$o12$g1$t1764979379$j59$l0$h0; _ga=GA1.1.992468134.1759064775; PHPSESSID=o58teels5igespn6nmj6gsqpe5; SESSID=nodo8" \
  --delay 1 \
  --threads 1

```