# Concentra't

Aplicació web Pomodoro per al personal PTGAS universitari.

## Ús

Obre `pomodoro-ptgas.html` directament al navegador. No cal servidor ni instal·lació.

## Característiques

- **Cronòmetre Pomodoro** amb cicle automàtic treball → pausa → treball
- **Registre d'activitat** per pomodoro: tema, origen, etiquetes i nota
- **Estadístiques** amb gràfics, mapa de calor i taula de sessions
- **Origens de sistema** (Whats, e-Admin) i origens d'usuari personalitzables
- **Etiquetes** amb color lliure, independents dels origens
- **Perfils de dades** per tenir múltiples espais de treball al mateix navegador
- **5 paletes de colors** incloent mode fosc i alt contrast (WCAG 2.2)
- **Exportació** CSV i JSON; importació amb mode fusió o substitució
- **Notificacions natives** i so de campana en acabar cada pomodoro
- **Objectiu diari** amb barra de progrés a la capçalera

## Tecnologia

Fitxer HTML únic autocontingut (HTML + CSS + JS). Sense framework, sense backend. Les dades es guarden al `localStorage` del navegador. L'única dependència externa és Google Fonts.

Compatible amb Chrome, Firefox i Edge moderns.
