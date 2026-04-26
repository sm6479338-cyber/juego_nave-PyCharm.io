# juego_nave-PyCharm.io
    # =============================================================================
    # JOC DELS METEORITS
    # =============================================================================
    # Un joc on una nau espacial ha d'esquivar meteorits que cauen del cel.
    # El jugador perd una vida cada vegada que un meteorit toca la nau.
    # Quan es perden totes les vides, apareix la pantalla de Game Over.
    #
    # FLUX DEL JOC:
    #   Pantalla d'inici  -->  (ESPAI)  -->  Joc  -->  (sense vides)  -->  Game Over
    #        ^                                                                  |
    #        |__________________________ (qualsevol tecla) ____________________|
    # =============================================================================
    
    import random
    
    import pygame
    from duplicity.commandline import select_files
    from pygame.locals import *
    
    
    # =============================================================================
    # CLASSE METEOR
    # Representa cada un dels meteorits que cauen per la pantalla.
    # =============================================================================
    class Meteor:

    def __init__(self, imatge, velocitat, pos_x, pos_y):
        """
        Constructor: s'executa automàticament quan creem un Meteor amb Meteor(...).
        Inicialitza tots els atributs (característiques) del meteorit.

        Paràmetres:
            imatge    -- ruta de la imatge del meteorit
            velocitat -- velocitat de caiguda inicial
            pos_x     -- posició horitzontal inicial
            pos_y     -- posició vertical inicial
        """
        # Guardem la ruta de la imatge
        self.imatge = imatge

        # Velocitat de caiguda (píxels per fotograma)
        self.velocitat = velocitat

        # Posició actual del meteorit
        self.x = pos_x
        self.y = pos_y

        # Carreguem la imatge des del disc
        self.img_meteor = pygame.image.load(self.imatge)
        self.img_inicio = pygame.image.load('assets/inicio.png')

        # El "rect" és el rectangle que envolta la imatge.
        # S'utilitza per detectar col·lisions i per dibuixar.
        # midbottom vol dir que (x, y) és el centre inferior del rectangle.
        self.rect_meteor = self.img_meteor.get_rect(midbottom=(self.x, self.y))

    def reiniciar(self):
        """
        Col·loca el meteorit a una posició aleatòria a la part superior
        de la pantalla, llest per tornar a caure.
        S'usa quan el meteorit surt per baix o quan colpeja la nau.
        """
        # Posició horitzontal aleatòria dins dels marges de la pantalla
        self.x = random.randint(30, 610)

        # Posem el meteorit just a sobre de la pantalla (y negatiu = invisible)
        self.y = -64

        # Velocitat aleatòria per fer el joc impredictible
        self.velocitat = random.randint(2, 5)                                        

    def moure(self):
        """
        Mou el meteorit cap avall un pas (segons la seva velocitat).
        Actualitza també el rectangle per mantenir-lo sincronitzat amb la posició.
        """
        # Cada fotograma, el meteorit baixa "velocitat" píxels
        self.y += self.velocitat

        # Actualitzem el rectangle perquè coincideixi amb la nova posició
        self.rect_meteor = self.img_meteor.get_rect(midbottom=(self.x, self.y))

    def ha_sortit_per_baix(self):
        """
        Comprova si el meteorit ha sortit per la part inferior de la pantalla.
        Retorna True si ha sortit, False si encara és visible.
        """
        return self.y >= 480

    def puntuar_i_reiniciar(self):
        """
        Si el meteorit ha sortit per baix, dona punts i el reinicia.
        Retorna els punts guanyats (5 si ha sortit, 0 si no).
        """
        if self.ha_sortit_per_baix():
            self.reiniciar()
            return 5  # El jugador guanya 5 punts per cada meteorit esquivat
        return 0  # Si no ha sortit, no hi ha punts


    # =============================================================================
    # CLASSE NAU
    # Representa la nau espacial controlada pel jugador.
    # =============================================================================
    class Nau:

    def __init__(self, imatge, velocitat, pos_x, pos_y, vides, imatge_vida):
        """
        Constructor de la nau.

        Paràmetres:
            imatge      -- ruta de la imatge de la nau
            velocitat   -- quants píxels es mou per tecla premuda
            pos_x       -- posició horitzontal inicial
            pos_y       -- posició vertical inicial
            vides       -- nombre de vides inicials
            imatge_vida -- ruta de la imatge de la icona de vida
        """
        self.imatge = imatge
        self.velocitat = velocitat
        #turbo
        self.velocitat_base = velocitat
        self.velocitat_turbo = velocitat *2
        self.turbo_activo = False
        self.trubo_duracion = 3000
        self.trubo_cooldown = 5000
        self.trubo_ini = 0
        self.trubo_fin = -self.trubo_cooldown
        #trubo-fin
        self.x = pos_x
        self.y = pos_y
        #escudo otra nave
        self.img_normal= pygame.image.load(self.imatge)
        self.img_escudo =pygame.image.load('assets/nave enemiga.png')

        tiempo_actual = pygame.time.get_ticks()

        #tiempo escudo
        self.escu_activo = False
        self.escu_duracion = 3000
        self.escu_cooldown = 8000
        self.tiempo_ini_escu = 0
        self.tiempo_ultimo_escu = -self.escu_cooldown
        self.tiempo_ultimo_escu = tiempo_actual


        # Carreguem les imatges
        self.img = pygame.image.load(self.imatge)
        self.img_vida = pygame.image.load(imatge_vida)

        # Rectangle de col·lisió i dibuix de la nau
        self.rect = self.img.get_rect(midbottom=(self.x, self.y))

        # Vides actuals i vides originals (per poder reiniciar)
        self.vides = vides
        self.vides_originals = vides

    def intentar_escudo(self):
        tiempo_actual = pygame.time.get_ticks()

        if not self.escu_activo:
            if tiempo_actual - self.tiempo_ultimo_escu >= self.escu_cooldown:
                self.escu_activo = True
                self.tiempo_ini_escu = tiempo_actual

    def reiniciar(self):
        """
        Torna la nau a la posició inicial i restaura totes les vides.
        S'usa quan comencem una partida nova.
        """
        self.x = 300
        self.y = 460
        self.rect = self.img.get_rect(midbottom=(self.x, self.y))
        self.vides = self.vides_originals

    def moure(self, pantalla_rect):
        """
        Llegeix les tecles premudes i mou la nau en la direcció corresponent.
        Impedeix que la nau surti dels límits de la pantalla.

        Paràmetre:
            pantalla_rect -- el rectangle de la pantalla, per limitar el moviment
        """
        # pygame.key.get_pressed() retorna una llista de True/False per cada tecla
        keys = pygame.key.get_pressed()


        # Moviment a l'esquerra (tecla A o fletxa esquerra)
        if keys[K_a] or keys[K_LEFT]:
            self.rect.x -= self.velocitat

        # Moviment a la dreta (tecla D o fletxa dreta)
        if keys[K_d] or keys[K_RIGHT]:
            self.rect.x += self.velocitat

        # Moviment cap amunt (tecla W o fletxa amunt)
        if keys[K_w] or keys[K_UP]:
            self.rect.y -= self.velocitat

        # Moviment cap avall (tecla S o fletxa avall)
        if keys[K_s] or keys[K_DOWN]:
            self.rect.y += self.velocitat


        # activaturbo
        tiempo_actual = pygame.time.get_ticks()
        keys = pygame.key.get_pressed()

        if (keys [K_LSHIFT] or keys [K_LSHIFT]) and not self.turbo_activo:
            if tiempo_actual - self.trubo_fin  >= self.trubo_cooldown:
                self.turbo_activo = True
                self.trubo_ini = tiempo_actual

        #control turbo
        if self.turbo_activo:
            if tiempo_actual - self.trubo_ini < self.trubo_duracion:
                self.velocitat = self.velocitat_turbo
            else:
                self.turbo_activo = False
                self.trubo_fin= tiempo_actual
                self.velocitat = self.velocitat_base
        else:
            self.velocitat = self.velocitat_base

        #escudo
        if keys [K_e] and not self.escu_activo:
            if tiempo_actual - self.tiempo_ultimo_escu >= self.escu_cooldown:
                self.escu_activo = True
                self.tiempo_ini_escu = tiempo_actual

        #se desactiva
        if self.escu_activo and tiempo_actual - self.tiempo_ini_escu >= self.escu_duracion:
            self.escu_activo = False
            self.tiempo_ini_escu = tiempo_actual

        #camio img
        if self.escu_activo:
            self.img = self.img_escudo
        else:
            self.img = self.img_normal


        # clamp_ip fa que la nau no pugui sortir dels límits de la pantalla
        self.rect.clamp_ip(pantalla_rect)

    def restar_vida(self):
        """
        Resta una vida a la nau.
        S'executa quan un meteorit colpeja la nau.
        """
    def hit(self):
        if not self.escu_activo:
            self.vides -= 1

    def esta_viva(self):
        """
        Comprova si la nau encara té vides.
        Retorna True si té almenys 1 vida, False si s'han acabat.
        """
        return self.vides > 0

    class disparos:
        def __init__(self, x, y):
            self.img = pygame.image.load('assets/disparos.png')
            self.rect = self.img.get_rect(midbottom=(x, y))
            self.velocitat = 8
        def moure(self):
            self.rect.y -= self.velocitat
        def dibuixar(self, pantalla):
            pantalla.blit(self.img, self.rect)
        def fora_pantalla(self):
            return self.rect.bottom < 0
    
    # =============================================================================
    # CLASSE JOC
    # Gestiona tota la lògica principal del joc: les pantalles, el bucle principal,
    # els dibuixos i les col·lisions.
    # =============================================================================
    class Joc:

    # Les pantalles possibles del joc (com etiquetes per saber on estem)
    PANTALLA_INICI    = "inici"
    PANTALLA_JOC      = "joc"
    PANTALLA_GAME_OVER = "game_over"

    def __init__(self, ample, alt, fps, nombre_meteors):
        """
        Constructor del joc. Inicialitza pygame i tots els elements necessaris.

        Paràmetres:
            ample          -- amplada de la finestra en píxels
            alt            -- alçada de la finestra en píxels
            fps            -- fotogrames per segon (velocitat del joc)
            nombre_meteors -- quants meteorits hi ha simultàniament
        """
        # Inicialitzem la llibreria pygame
        pygame.init()

        # Guardem la configuració bàsica
        self.ample = ample
        self.alt = alt
        self.fps = fps
        self.numero_meteors = nombre_meteors

        # Creem la finestra del joc
        self.pantalla = pygame.display.set_mode((self.ample, self.alt))
        pygame.display.set_caption("Joc dels Meteorits")

        # El rellotge controla la velocitat del joc (FPS)
        self.rellotge = pygame.time.Clock()

        # Llista buida de meteorits (es creen a preparar_partida)
        self.meteors = []
        self.disparos = []

        # Puntuació del jugador
        self.punts = 0

        # Creem la nau del jugador
        self.nau = Nau('assets/nave amiga.png', velocitat=5, pos_x=300, pos_y=450,
                       vides=3, imatge_vida='assets/vida.png')

        # Pantalla activa al començar (la presentació)
        self.pantalla_activa = self.PANTALLA_INICI

    # -------------------------------------------------------------------------
    # GESTIÓ DE PANTALLES
    # -------------------------------------------------------------------------

    def iniciar_joc(self):
        """
        Bucle principal del programa.
        Decideix quina pantalla mostrar en cada moment segons l'estat del joc.
        S'executa indefinidament fins que l'usuari tanca la finestra.
        """
        while True:
            if self.pantalla_activa == self.PANTALLA_INICI:
                self.mostrar_pantalla_inici()

            elif self.pantalla_activa == self.PANTALLA_JOC:
                self.preparar_partida()
                self.mostrar_pantalla_joc()

            elif self.pantalla_activa == self.PANTALLA_GAME_OVER:
                self.mostrar_pantalla_game_over()

    def mostrar_pantalla_inici(self):
        inicio = pygame.image.load('assets/inicio.png')
        self.pantalla.blit(inicio, (0,0))
        """
        Mostra la pantalla de presentació del joc.
        Quan el jugador prem ESPAI, passa a la pantalla de joc.
        """
        # Fons de color magenta (pots posar una imatge aquí si en tens una)
        self.pantalla.blit(inicio, (0, 0))

        # Escrivim el títol i les instruccions
        self._dibuixar_text("Juego Meteoritos", mida=64, color=(255, 255, 255),
                             x=self.ample // 2, y=160, centrat=True)
        self._dibuixar_text("Prem ESPAI per començar", mida=28, color=(198, 198, 198),
                             x=self.ample // 2, y=280, centrat=True)
        self._dibuixar_text("Mou la nau amb les fletxes o WASD", mida=22,
                             color=(198, 198, 198), x=self.ample // 2, y=330, centrat=True)

        pygame.display.update()

        # Bucle d'espera: restem aquí fins que es premi ESPAI o es tanqui la finestra
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                pygame.quit()
                exit()
            if event.type == pygame.KEYDOWN and event.key == K_SPACE:
                # Canviem l'estat a "joc" per passar a la partida
                self.pantalla_activa = self.PANTALLA_JOC

        # Controlem els FPS fins i tot en la pantalla d'inici
        self.rellotge.tick(self.fps)

    def mostrar_pantalla_game_over(self):
        """
        Mostra la pantalla de Game Over amb la puntuació final.
        Quan el jugador prem qualsevol tecla, torna a la pantalla d'inici.
        """
        # Carreguem i mostrem la imatge de Game Over
        img_game_over = pygame.image.load('assets/gam_over.png')
        self.pantalla.blit(img_game_over, (0, 0))

        # Mostrem la puntuació final per sobre de la imatge
        self._dibuixar_text(f"Puntuació: {self.punts}", mida=40, color=(255, 0, 0),
                             x=self.ample // 2, y=380, centrat=True)
        self._dibuixar_text("Prem qualsevol tecla per tornar", mida=30,
                             color=(255, 0, 0), x=self.ample // 2, y=430, centrat=True)

        pygame.display.update()

        # Esperem que es premi una tecla o es tanqui la finestra
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                pygame.quit()
                exit()
            if event.type == pygame.KEYDOWN:
                # Tornem a la pantalla d'inici
                self.pantalla_activa = self.PANTALLA_INICI

        self.rellotge.tick(self.fps)

    def mostrar_pantalla_joc(self):
        juego = pygame.image.load('assets/fondo juego.png')
        self.pantalla.fill((0, 0, 0))
        """
        Bucle principal de la partida.
        S'executa fotograma a fotograma mentre el jugador té vides.
        Quan es queden sense vides, canvia a la pantalla de Game Over.
        """

        # Aquest bucle s'executa mentre la pantalla activa sigui "joc"
        while self.pantalla_activa == self.PANTALLA_JOC:

            # 1. Gestionem els events (teclat, ratolí, tancar finestra...)
            self._gestionar_events()

            # 2. Omplim el fons de negre (esborrem el fotograma anterior)
            self.pantalla.blit(juego, (0, 0))

            # 3. Actualitzem la posició de tots els elements
            self.nau.moure(self.pantalla.get_rect())
            self._moure_meteors()

            for disparos in self.disparos[:]:
                disparos.moure()
                if disparos.fora_pantalla():
                    self.disparos.remove(disparos)
            # 4. Comprovem si algun meteorit ha xocat amb la nau
            self._control_colisions()
            for disparos in self.disparos:
                disparos.dibuixar(self.pantalla)
            # 5. Dibuixem tots els elements a la pantalla
            self._dibuixar_meteors()
            self._dibuixar_nau()
            self._dibuixar_vides()
            self._dibuixar_punts()


            # 6. Si la nau no té vides, anem a Game Over
            if not self.nau.esta_viva():
                self.pantalla_activa = self.PANTALLA_GAME_OVER

            # 7. Mostrem el fotograma complet i esperem per mantenir els FPS
            pygame.display.update()
            self.rellotge.tick(self.fps)

    # -------------------------------------------------------------------------
    # PREPARACIÓ DE LA PARTIDA
    # -------------------------------------------------------------------------

    def preparar_partida(self):
        """
        Reinicia tots els elements per començar una partida nova:
        - Esborra i crea els meteorits
        - Reinicia la nau a la posició inicial
        - Posa la puntuació a zero
        """
        # Buidem la llista de meteorits anterior
        self.meteors.clear()
        self.disparos.clear()

        # Creem tants meteorits com indiqui numero_meteors
        for i in range(self.numero_meteors):
            meteor_nou = Meteor('assets/meteorito.png', velocitat=0, pos_x=0, pos_y=0)
            meteor_nou.reiniciar()  # Col·loquem-lo a una posició aleatòria inicial
            self.meteors.append(meteor_nou)

        # Reiniciem la nau (posició i vides)
        self.nau.reiniciar()

        # Reiniciem la puntuació
        self.punts = 0

    # -------------------------------------------------------------------------
    # LÒGICA DEL JOC (mètodes privats, s'usen des de mostrar_pantalla_joc)
    # -------------------------------------------------------------------------

    def _gestionar_events(self):
        """
        Processa els events de pygame: tancar finestra, tecles, etc.
        Els mètodes que comencen amb _ són "privats": s'usen internament.
        """
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                pygame.quit()
                exit()
            if event.type == KEYDOWN:
                if event.key == K_SPACE:
                    dispar = disparos(self.nau.rect.centerx, self.nau.rect.top)
                    self.disparos.append(dispar)

            if event.type == KEYDOWN:
                if event.key == K_e:
                    self.nau.intentar_escudo()

    def _moure_meteors(self):
        """
        Mou tots els meteorits cap avall i suma punts si algun surt per baix.
        """
        for meteor in self.meteors:
            meteor.moure()
            # Si el meteorit surt per baix, puntuar_i_reiniciar el reinicia i retorna punts
            self.punts += meteor.puntuar_i_reiniciar()

    def _control_colisions(self):
        """
        Comprova si algun meteorit toca la nau.
        Si hi ha col·lisió: el meteorit es reinicia i la nau perd una vida.
        """
        for meteor in self.meteors:
            # colliderect comprova si dos rectangles es superposen
            if meteor.rect_meteor.colliderect(self.nau.rect):
                meteor.reiniciar()      # El meteorit torna a dalt
                self.nau.restar_vida()  # La nau perd una vida
        for dispar in self.disparos[:]:
            for meteor in self.meteors:
                if dispar.rect.colliderect(meteor.rect_meteor):
                    meteor.reiniciar()
                    if dispar in self.disparos:
                        self.disparos.remove(dispar)
                    self.punts += 10
                    break

    # -------------------------------------------------------------------------
    # DIBUIX DELS ELEMENTS
    # -------------------------------------------------------------------------

    def _dibuixar_meteors(self):
        """
        Dibuixa tots els meteorits a la pantalla en la seva posició actual.
        """
        for meteor in self.meteors:
            # blit dibuixa una imatge sobre la pantalla
            self.pantalla.blit(meteor.img_meteor, meteor.rect_meteor)

    def _dibuixar_nau(self):
        """
        Dibuixa la nau a la pantalla en la seva posició actual.
        """
        self.pantalla.blit(self.nau.img, self.nau.rect)

    def _dibuixar_vides(self):
        """
        Dibuixa les icones de vida a la cantonada superior dreta.
        Una icona per cada vida que queda.
        """
        # Posicions horitzontals on posem cada icona de vida
        posicions_x = [500, 540, 580]

        # Dibuixem tantes icones com vides té la nau
        for i in range(self.nau.vides):
            if i < len(posicions_x):
                self.pantalla.blit(self.nau.img_vida, (posicions_x[i], 20))

    def _dibuixar_punts(self):
        """
        Dibuixa la puntuació actual a la cantonada superior esquerra.
        """
        self._dibuixar_text(str(self.punts), mida=32, color=(255, 255, 255),
                             x=140, y=30, centrat=False)

    def _dibuixar_text(self, text, mida, color, x, y, centrat=False):
        """
        Funció auxiliar per dibuixar text a la pantalla.

        Paràmetres:
            text    -- el text a mostrar
            mida    -- mida de la lletra en punts
            color   -- color en format RGB, p.ex. (255, 255, 255) per blanc
            x, y    -- posició on dibuixar
            centrat -- si True, centra el text horitzontalment a partir de x.
        """
        font = pygame.font.SysFont(None, mida)
        imatge_text = font.render(text, True, color)

        if centrat:
            # Calculem la posició per centrar el text
            rect_text = imatge_text.get_rect(center=(x, y))
            self.pantalla.blit(imatge_text, rect_text)
        else:
            self.pantalla.blit(imatge_text, (x, y))


    # =============================================================================
    # INICI DEL PROGRAMA
    # =============================================================================
    # Tot el que hi ha aquí a baix s'executa quan arranquem el fitxer.
    # Creem el joc i el posem en marxa.
    
    partida = Joc(
        ample=640,
        alt=480,
        fps=60,
        nombre_meteors=4
    )
    partida.iniciar_joc()
