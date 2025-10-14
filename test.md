from abc import *
from board import Board
import math


class State(object):
    """
    Apstraktna klasa koja opisuje stanje pretrage.
    """

    @abstractmethod
    def init(self, board: Board, parent=None, position=None, goal_position=None, action=None):
        """
        :param board: Board - tabla
        :param parent: State - roditeljsko stanje
        :param position: (int x, int y) - pozicija stanja
        :param goal_position: (int x, int y) - pozicija krajnjeg stanja
        :return:
        """
        self.board = board  # Reference na stanje table koje se vidi na ekranu. Ovo se ne menja
        self.parent = parent  # roditeljsko stanje
        self.action = action # akcija koja je dovela do trenutnog stanja


        if self.parent is None:  # ako nema roditeljsko stanje, onda je ovo inicijalno stanje
            # pronaladji elemente sa table
            self.position = board.find_position(self.get_agent_code())  # pronadji pocetnu poziciju
            self.goal_position = board.find_position(self.get_agent_goal_code())  # pronadji krajnju poziciju
            self.checkpoints = tuple(board.find_all_positions(self.get_checkpoint_code()))
            self.teleports = tuple(board.find_all_positions(self.get_teleport_code()))
            self.teleport = False
            self.fire = board.find_position(self.get_fire_code())
        else:  # ako ima roditeljsko stanje, samo sacuvaj vrednosti parametara
            self.position = position
            self.goal_position = goal_position
            self.checkpoints = self.parent.checkpoints
            self.teleports = self.parent.teleports
            self.teleport = self.parent.teleport
            self.fire = self.parent.fire

        self.depth = parent.depth + 1 if parent is not None else 1  # povecaj dubinu/nivo pretrage

    def get_next_states(self):
        new_positions = self.get_legal_positions()  # dobavi moguce (legalne) sledece pozicije iz trenutne pozicije
        next_states = []
        # napravi listu mogucih sledecih stanja na osnovu mogucih sledecih pozicija
        for new_position, action in new_positions:
            next_state = self.class(self.board, self, new_position, self.goal_position, action)
            next_states.append(next_state)
        return next_states


    def get_agent_code(self):
        return 'r'

    def get_agent_goal_code(self):
        return 'g'
    
    def get_checkpoint_code(self):
        return 'b'
    
    def get_teleport_code(self):
        return 'y'

    def get_fire_code(self):
        return 'f'

    @abstractmethod
    def get_legal_positions(self):
        """
        Apstraktna metoda koja treba da vrati moguce (legalne) sledece pozicije na osnovu trenutne pozicije.
        :return: list
        """
        pass

    @abstractmethod
    def is_final_state(self):
        """
        Apstraktna metoda koja treba da vrati da li je treuntno stanje zapravo zavrsno stanje.
        :return: bool
        """
        pass

    @abstractmethod
    def unique_hash(self):
        """
        Apstraktna metoda koja treba da vrati string koji je JEDINSTVEN za ovo stanje
        (u odnosu na ostala stanja).
        :return: str
        """
        pass
    
    @abstractmethod
    def get_cost_estimate(self):
        """
        Apstraktna metoda koja treba da vrati procenu cene
        (vrednost heuristicke funkcije - h(n)) za ovo stanje.
        Koristi se za vodjene pretrage.
        :return: float
        """
        pass
    
    @abstractmethod
    def get_current_cost(self):
        """
        Apstraktna metoda koja treba da vrati stvarnu dosadašnju trenutnu cenu za ovo stanje, odnosno g(n)
        Koristi se za vodjene pretrage.
        :return: float
        """
        pass


class RobotState(State):
def init(self, board: Board, parent: State=None, position=None, goal_position=None, action=None):
        super().init(board, parent, position, goal_position, action)
        # posle pozivanja super konstruktora, mogu se dodavati "custom" stvari vezani za stanje
        if self.parent is None:
            self.cost = 0
            # Sortiramo checkpoint-e po x koordinati, desni (vece x) je prvi
            self.checkpoints = tuple(sorted(list(self.checkpoints), key=lambda c: c[1], reverse=True))
        else:
            self.cost = self.parent.cost + 1
            
            # Proveravamo da li smo pokupili checkpoint
            if self.position in self.parent.checkpoints:
                # Ako jesmo, izbacujemo ga iz liste checkpoint-a za sledece stanje
                new_checkpoints = list(self.parent.checkpoints)
                new_checkpoints.remove(self.position)
                self.checkpoints = tuple(new_checkpoints)
        

    def get_legal_positions(self):
        
        actions = [(0, 1), (0, -1), (1, 0), (-1, 0)]
        # akcije sahovskoh konja
        knight_actions = [(1, 2), (1, -2), (-1, 2), (-1, -2), (2, 1), (2, -1), (-2, 1), (-2, -1)]

        # Ako je ostao samo jedan checkpoint (levi), krecemo se kao konj
        if len(self.checkpoints) == 1:
            current_actions = knight_actions
        else:
            current_actions = actions

        row, col = self.position 
        new_positions = []
        for d_row, d_col in current_actions: 
            new_row = row + d_row
            new_col = col + d_col 

            if not self.board.is_out_of_bounds(new_row, new_col) and not self.board.hits_wall(new_row, new_col):
                new_positions.append(((new_row, new_col), (d_row, d_col)))
        return new_positions

    def is_final_state(self):
        # Finalno stanje je kada smo na ciljnoj poziciji i pokupili smo sve checkpoint-e
        return self.position == self.goal_position and len(self.checkpoints) == 0

    def unique_hash(self):
        # Za jedinstvenost stanja koristimo poziciju i preostale checkpoint-e
        return str(self.position) + str(self.checkpoints)
    
    def get_cost_estimate(self):
        # Heuristika za A*
        if len(self.checkpoints) > 0:
            # Ako ima preostalih checkpoint-a, heuristika je udaljenost do sledeceg
            return self.manhattan_distance(self.position, self.checkpoints[0])
        else:
            # Ako nema, heuristika je udaljenost do cilja
            return self.manhattan_distance(self.position, self.goal_position)
        
    def get_current_cost(self):
        return self.cost

    
    def manhattan_distance(self, pointA, pointB):
        return abs(pointA[0] - pointB[0]) + abs(pointA[1] - pointB[1])
    
    def euclidian_distance(self, pointA, pointB):
        return math.sqrt((pointA[0] - pointB[0])**2 + (pointA[1] - pointB[1])**2)
    
    def diagonal_distance(self, pointA, pointB):
        return max([abs(pointA[0] - pointB[0]), abs(pointA[1] - pointB[1])])


    # dodajemo da lakse debagujemo
    def repr(self):
        return f'RobotState(pos={self.position}, depth={self.depth})'


# Definišemo pomoćnu funkciju van klase da bi se izbegla lambda funkcija
def uzmi_red(tacka):
    """Pomoćna funkcija koja vraća prvu koordinatu (red) tačke. Koristi se za sortiranje od gore ka dole."""
    return tacka[0]

class CustomRobot1(State):

def init(self, board: Board, parent: State=None, position=None, goal_position=None, action=None):
        super().init(board, parent, position, goal_position, action)
        
        if self.parent is None:
            self.cost = 0
            
            # --- Podela table na levu i desnu stranu ---
            # Pronalazimo vertikalnu sredinu table. Sve kolone sa indeksom >= mid_col su desno.
            mid_col = self.board.cols // 2
            
            # --- Komentar za podelu na GORNJU i DONJU stranu ---
            # Ako bi zadatak zahtevao da se prvo sakupe kutije sa gornje, pa sa donje strane,
            # logika bi bila analogna. Umesto deljenja po kolonama, delili bismo po redovima:
            # 1. Našli bismo horizontalnu sredinu: mid_row = self.board.rows // 2
            # 2. Podelili bismo kutije na gornje (one sa redom < mid_row) i donje (red >= mid_row).
            #    upper_checkpoints = [c for c in all_checkpoints if c[0] < mid_row]
            #    lower_checkpoints = [c for c in all_checkpoints if c[0] >= mid_row]
            # 3. Redosled bi bio: self.checkpoints = upper_checkpoints + lower_checkpoints
            
            all_checkpoints = list(self.checkpoints)
            right_checkpoints = [c for c in all_checkpoints if c[1] >= mid_col]
            left_checkpoints = [c for c in all_checkpoints if c[1] < mid_col]

            # Sortiramo obe liste od gore ka dole (po rastućem broju reda).
            # Koristimo uzmi_red da kažemo sort metodi po kom elementu da sortira.
            right_checkpoints.sort(key=uzmi_red)
            left_checkpoints.sort(key=uzmi_red)

            # Postavljamo redosled checkpoint-a: prvo desni, pa levi.
            # self.checkpoints je sada lista, što pojednostavljuje kasnije operacije.
            self.checkpoints = right_checkpoints + left_checkpoints
            self.num_right_checkpoints = len(right_checkpoints)
        else:
            self.cost = self.parent.cost + 1
            # Preuzimamo vrednosti od roditelja
            self.num_right_checkpoints = self.parent.num_right_checkpoints
            self.checkpoints = list(self.parent.checkpoints) # Kreiramo kopiju da ne menjamo stanje roditelja
            
            # Proveravamo da li smo pokupili checkpoint i uklanjamo ga iz liste
            if self.position in self.checkpoints:
                self.checkpoints.remove(self.position)
        

    def get_legal_positions(self):
        # --- Potezi za druge šahovske figure (primeri) ---
        # Kralj (potezi u svim smerovima za jedno polje)
        # king_actions = [(0, 1), (0, -1), (1, 0), (-1, 0), (1, 1), (1, -1), (-1, 1), (-1, -1)]

        # Top (horizontalni i vertikalni potezi)
        # rook_actions = []
        # row, col = self.position
        # directions = [(1, 0), (-1, 0), (0, 1), (0, -1)]  # Dole, Gore, Desno, Levo
        # for dr, dc in directions:
        #     for i in range(1, max(self.board.rows, self.board.cols)):
        #         new_row, new_col = row + i * dr, col + i * dc
        #         if self.board.is_out_of_bounds(new_row, new_col) or self.board.hits_wall(new_row, new_col):
        #             break  # Prekini kretanje u ovom pravcu
        #         rook_actions.append(((new_row, new_col), (dr, dc)))

        # Lovac (dijagonalni potezi)
        # bishop_actions = []
        # row, col = self.position
        # directions = [(1, 1), (1, -1), (-1, 1), (-1, -1)]  # Dijagonalno
        # for dr, dc in directions:
        #     for i in range(1, max(self.board.rows, self.board.cols)):
        #         new_row, new_col = row + i * dr, col + i * dc
        #         if self.board.is_out_of_bounds(new_row, new_col) or self.board.hits_wall(new_row, new_col):
        #             break  # Prekini kretanje u ovom pravcu
        #         bishop_actions.append(((new_row, new_col), (dr, dc)))

        # Kraljica = potezi Topa + potezi Lovca
        # queen_actions = rook_actions + bishop_actions

actions = [(0, 1), (0, -1), (1, 0), (-1, 0)]
        knight_actions = [(1, 2), (1, -2), (-1, 2), (-1, -2), (2, 1), (2, -1), (-2, 1), (-2, -1)]

        # Proveravamo da li su svi desni checkpoint-i pokupljeni.
        # To znamo ako je broj preostalih checkpoint-a manji ili jednak broju levih.
        # (Ukupno - Desni = Levi)
        num_left_to_collect = len(self.checkpoints)
        num_left_total = len(self.checkpoints) - self.num_right_checkpoints
        
        # --- Komentar za GORE/DOLE ---
        # Ovde bi uslov bio da li su svi gornji checkpoint-i pokupljeni.
        # Ako jesu, i ako je ostalo još (donjih) checkpoint-a, prelazimo na kretanje konja.
        # if len(self.checkpoints) <= num_lower_checkpoints and len(self.checkpoints) > 0:
        #     current_actions = knight_actions
        
        if num_left_to_collect <= num_left_total and num_left_to_collect > 0:
            current_actions = knight_actions
        else:
            current_actions = actions

        row, col = self.position 
        new_positions = []
        for d_row, d_col in current_actions: 
            new_row = row + d_row
            new_col = col + d_col 

            if not self.board.is_out_of_bounds(new_row, new_col) and not self.board.hits_wall(new_row, new_col):
                new_positions.append(((new_row, new_col), (d_row, d_col)))
        return new_positions

    def is_final_state(self):
        return self.position == self.goal_position and len(self.checkpoints) == 0

    def unique_hash(self):
        # Za jedinstvenost stanja, poziciju kombinujemo sa string reprezentacijom liste checkpoint-a.
        return str(self.position) + str(self.checkpoints)
    
    def get_cost_estimate(self):
        # Heuristika za A*
        if len(self.checkpoints) > 0:
            # Ako ima preostalih checkpoint-a, heuristika je udaljenost do sledeceg
            # plus broj preostalih checkpointa da bi se favorizovalo sakupljanje
            return self.manhattan_distance(self.position, self.checkpoints[0]) + len(self.checkpoints)
        else:
            # Ako nema, heuristika je udaljenost do cilja
            return self.manhattan_distance(self.position, self.goal_position)
        
    def get_current_cost(self):
        return self.cost

    
    def manhattan_distance(self, pointA, pointB):
        return abs(pointA[0] - pointB[0]) + abs(pointA[1] - pointB[1])
    
    def euclidian_distance(self, pointA, pointB):
        return math.sqrt((pointA[0] - pointB[0])**2 + (pointA[1] - pointB[1])**2)
    
    def diagonal_distance(self, pointA, pointB):
        return max([abs(pointA[0] - pointB[0]), abs(pointA[1] - pointB[1])])


    # dodajemo da lakse debagujemo
    def repr(self):
        return f'CustomRobot1(pos={self.position}, depth={self.depth}, ckpts={len(self.checkpoints)})'


class CustomRobot2(State):
    """
    Agent koji se kreće što dalje od vatre.
    Koristi A* sa heuristikom koja kombinuje udaljenost do cilja i udaljenost od vatre.
    Nasleđuje State i implementira sve potrebne apstraktne metode.
    """
    def init(self, board: Board, parent: State=None, position=None, goal_position=None, action=None):
        super().init(board, parent, position, goal_position, action)
        if self.parent is None:
            self.cost = 0
            # Osiguravamo da je checkpoints lista ako je ovo početno stanje
            self.checkpoints = list(self.checkpoints)
        else:
            self.cost = self.parent.cost + 1
            # Kreiramo kopiju liste od roditelja
            self.checkpoints = list(self.parent.checkpoints)
            
            # Za razliku od RobotState, ne radimo ništa specijalno sa checkpoint-ima ovde,
            # ali zadržavamo logiku za slučaj da se koristi mapa sa njima.
            if self.position in self.checkpoints:
                self.checkpoints.remove(self.position)

def get_legal_positions(self):
        actions = [(0, 1), (0, -1), (1, 0), (-1, 0)]
        row, col = self.position 
        new_positions = []
        for d_row, d_col in actions: 
            new_row = row + d_row
            new_col = col + d_col 

            if not self.board.is_out_of_bounds(new_row, new_col) and not self.board.hits_wall(new_row, new_col):
                new_positions.append(((new_row, new_col), (d_row, d_col)))
        return new_positions

    def is_final_state(self):
        # Finalno stanje je kada smo na ciljnoj poziciji i pokupili smo sve checkpoint-e
        return self.position == self.goal_position and len(self.checkpoints) == 0

    def unique_hash(self):
        # Za jedinstvenost stanja koristimo poziciju i preostale checkpoint-e
        return str(self.position) + str(self.checkpoints)

    def get_current_cost(self):
        return self.cost

    def manhattan_distance(self, pointA, pointB):
        return abs(pointA[0] - pointB[0]) + abs(pointA[1] - pointB[1])

    def get_cost_estimate(self):
        # Osnovna heuristika je udaljenost do sledećeg cilja (checkpoint ili goal)
        if len(self.checkpoints) > 0:
            h_goal = self.manhattan_distance(self.position, self.checkpoints[0])
        else:
            h_goal = self.manhattan_distance(self.position, self.goal_position)

        # Heuristika za izbegavanje vatre
        # Ako vatra postoji na mapi
        if self.fire:
            dist_to_fire = self.manhattan_distance(self.position, self.fire)
            
            # Želimo da je udaljenost od vatre što veća.
            # Pošto A* minimizuje vrednost heuristike, dodajemo kaznu za blizinu vatre.
            # Koristimo inverznu vrednost udaljenosti - što je robot bliže vatri,
            # to je dist_to_fire manja, a 1 / dist_to_fire veća.
            # Dodajemo 1 da izbegnemo deljenje nulom ako je robot na polju vatre.
            # Množimo sa konstantom (npr. 100) da pojačamo uticaj vatre.
            h_fire = 100 / (dist_to_fire + 1)
        else:
            # Ako nema vatre, ovaj deo heuristike je 0.
            h_fire = 0
        
        # Ukupna heuristika je zbir heuristike ka cilju i kazne za blizinu vatre.
        # Agent će birati putanje koje minimizuju ovu vrednost, tj. idu ka cilju i dalje od vatre.
        return h_goal + h_fire

    def repr(self):
        return f'CustomRobot2(pos={self.position}, depth={self.depth})'




