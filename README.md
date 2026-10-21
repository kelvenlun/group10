# group10
貪吃蛇遊戲
import pygame
import random
import sys
from collections import deque  # 引入隊列，用於 BFS 尋路演算法

# ==========================================
# 1. 遊戲初始化與常數設定
# ==========================================
pygame.init()
WIDTH, HEIGHT = 800, 600
GRID_SIZE = 20
COLS, ROWS = WIDTH // GRID_SIZE, HEIGHT // GRID_SIZE

# 顏色定義
BLACK = (20, 20, 20)
GRID_COLOR = (40, 40, 40)
GREEN = (0, 200, 0)
LIGHT_GREEN = (50, 255, 50)
RED = (200, 0, 0)
LIGHT_RED = (255, 50, 50)
YELLOW = (255, 215, 0)
CYAN = (0, 255, 255)
WHITE = (255, 255, 255)
GRAY = (100, 100, 100)
HOVER_GRAY = (150, 150, 150)
BLUE = (50, 100, 255)       # 護盾道具顏色
SHIELD_AURA = (100, 200, 255) # 護盾無敵發光顏色

screen = pygame.display.set_mode((WIDTH, HEIGHT))
pygame.display.set_caption("貪吃蛇大逃殺：極速迴廊 - 第十組 (進階版)")
clock = pygame.time.Clock()

font_small = pygame.font.Font(None, 36)
font_large = pygame.font.Font(None, 64)

STATE_MENU = 0
STATE_PLAYING = 1
STATE_GAMEOVER = 2
STATE_WIN = 3

def draw_grid():
    for x in range(0, WIDTH, GRID_SIZE):
        pygame.draw.line(screen, GRID_COLOR, (x, 0), (x, HEIGHT))
    for y in range(0, HEIGHT, GRID_SIZE):
        pygame.draw.line(screen, GRID_COLOR, (0, y), (WIDTH, y))

def draw_button(text, x, y, w, h, mouse_pos):
    rect = pygame.Rect(x, y, w, h)
    is_hovered = rect.collidepoint(mouse_pos)
    color = HOVER_GRAY if is_hovered else GRAY
    pygame.draw.rect(screen, color, rect, border_radius=10)
    pygame.draw.rect(screen, WHITE, rect, 2, border_radius=10)
    text_surf = font_small.render(text, True, WHITE)
    text_rect = text_surf.get_rect(center=rect.center)
    screen.blit(text_surf, text_rect)
    return rect, is_hovered

# --- 進階 AI 演算法：BFS 尋路 ---
# --- 進階 AI 演算法：BFS 尋路 ---
def get_bfs_next_step(start, target, obstacles):
    """利用 BFS (廣度優先搜尋) 找到前往玩家的最短路徑第一步"""
    queue = deque([(start, [])])
    visited = set()
    visited.add(tuple(start))
    
    while queue:
        curr, path = queue.popleft()
        if curr == target:
            return path[0] if path else start
            
        x, y = curr
        neighbors = [((x+1)%COLS, y), ((x-1)%COLS, y), (x, (y+1)%ROWS), (x, (y-1)%ROWS)]
        
        for nx, ny in neighbors:
            # 💡 修正這裡：如果下一步是我們的「目標」，就算它被列在障礙物裡也要允許踩上去！
            if (nx, ny) not in visited and ([nx, ny] not in obstacles or [nx, ny] == target):
                visited.add((nx, ny))
                queue.append(([nx, ny], path + [[nx, ny]]))
                
    return None # 如果被死路卡住，回傳 None
def init_game_vars():
    player_snake = [[10, 10], [9, 10], [8, 10]]
    player_dir = (1, 0)
    
    enemy_snakes = [
        [[COLS-5, 5], [COLS-4, 5], [COLS-3, 5], [COLS-2, 5]],
        [[5, ROWS-5], [4, ROWS-5], [3, ROWS-5], [2, ROWS-5]]
    ]
    
    food = [random.randint(1, COLS-2), random.randint(1, ROWS-2)]
    # 隨機產生護盾道具
    shield_item = [random.randint(1, COLS-2), random.randint(1, ROWS-2)]
    
    score = 0
    exit_pos = None
    enemy_move_counter = 0
    
    # 新增：進階功能變數
    has_shield = False
    invincible_timer = 0
    dash_speed_frames = 0  # 衝刺加速計時器
    
    return player_snake, player_dir, enemy_snakes, food, shield_item, score, exit_pos, enemy_move_counter, has_shield, invincible_timer, dash_speed_frames

def main():
    state = STATE_MENU
    move_delay = 180  
    last_move_time = 0

    player_snake, player_dir, enemy_snakes, food, shield_item, score, exit_pos, enemy_move_counter, has_shield, invincible_timer, dash_speed_frames = init_game_vars()
    target_score = 5

    while True:
        mouse_pos = pygame.mouse.get_pos()
        mouse_clicked = False
        force_move = False # 用於觸發「瞬間衝刺」

        # ==========================================
        # 2. 事件與操作處理
        # ==========================================
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                pygame.quit()
                sys.exit()
            if event.type == pygame.MOUSEBUTTONDOWN and event.button == 1:
                mouse_clicked = True
            
            if event.type == pygame.KEYDOWN:
                if event.key == pygame.K_ESCAPE:
                    pygame.quit()
                    sys.exit()
                
                if state == STATE_PLAYING:
                    # 轉向
                    if (event.key == pygame.K_UP or event.key == pygame.K_w) and player_dir != (0, 1):
                        player_dir = (0, -1)
                    elif (event.key == pygame.K_DOWN or event.key == pygame.K_s) and player_dir != (0, -1):
                        player_dir = (0, 1)
                    elif (event.key == pygame.K_LEFT or event.key == pygame.K_a) and player_dir != (1, 0):
                        player_dir = (-1, 0)
                    elif (event.key == pygame.K_RIGHT or event.key == pygame.K_d) and player_dir != (-1, 0):
                        player_dir = (1, 0)
                        
                    # ====== 核心功能 1：空白鍵脫皮衝刺 ======
                    # 條件：按下空白鍵且長度 > 3 (至少有吃到過東西或有足夠節數)
                    if event.key == pygame.K_SPACE and len(player_snake) > 3:
                        player_snake.pop() # 消耗一節尾巴
                        if score > 0: score -= 1 # 扣除一把鑰匙的進度
                        dash_speed_frames = 15 # 接下來的 15 步會變成極速衝刺！

        # ==========================================
        # 3. 遊戲邏輯與時間控制
        # ==========================================
        screen.fill(BLACK)
        draw_grid()

        if state == STATE_MENU:
            title_text = font_large.render("SNAKE ESCAPE", True, LIGHT_GREEN)
            screen.blit(title_text, (WIDTH//2 - title_text.get_width()//2, HEIGHT//2 - 100))
            subtitle = font_small.render("Group 10 Project", True, WHITE)
            screen.blit(subtitle, (WIDTH//2 - subtitle.get_width()//2, HEIGHT//2 - 40))

            btn_rect, is_hovered = draw_button("START GAME", WIDTH//2 - 100, HEIGHT//2 + 40, 200, 50, mouse_pos)
            if mouse_clicked and is_hovered:
                player_snake, player_dir, enemy_snakes, food, shield_item, score, exit_pos, enemy_move_counter, has_shield, invincible_timer, dash_speed_frames = init_game_vars()
                state = STATE_PLAYING

        elif state == STATE_PLAYING:
            
            # 無敵計時器遞減 (每偵減1)
            if invincible_timer > 0:
                invincible_timer -= 1

            # 計算當下的移動延遲 (如果有衝刺 buff，延遲縮短到 30 毫秒極速)
            current_delay = 30 if dash_speed_frames > 0 else max(40, 180 - (score * 20))
            current_time = pygame.time.get_ticks()

            if current_time - last_move_time > current_delay:
                if dash_speed_frames > 0:
                    dash_speed_frames -= 1 # 消耗衝刺步數
                
                # --- 玩家移動 ---
                new_x = (player_snake[0][0] + player_dir[0]) % COLS
                new_y = (player_snake[0][1] + player_dir[1]) % ROWS
                new_head = [new_x, new_y]
                
                player_snake.insert(0, new_head)
                
                # --- 物品拾取 ---
                # 1. 吃鑰匙 (食物)
                if new_head == food:
                    score += 1
                    if score < target_score:
                        while True:
                            food = [random.randint(1, COLS-2), random.randint(1, ROWS-2)]
                            if food not in player_snake and food != shield_item:
                                break
                    else:
                        food = [-1, -1] 
                        exit_pos = [COLS // 2, 0]
                else:
                    player_snake.pop()
                
                # 2. 吃護盾道具
                if new_head == shield_item:
                    has_shield = True
                    shield_item = [-1, -1] # 吃掉後從地圖移除

                if exit_pos and new_head == exit_pos:
                    state = STATE_WIN

                # --- 敵方蛇 AI 移動 (BFS 尋路) ---
                enemy_move_counter += 1
                if enemy_move_counter % 3 != 0:
                    # 整理出所有的障礙物：包含玩家身體、以及另一條敵方蛇的身體
                    all_obstacles = player_snake.copy()
                    
                    for e_snake in enemy_snakes:
                        e_head = e_snake[0]
                        
                        # ====== 核心功能 3：BFS 智慧追擊 ======
                        next_step = get_bfs_next_step(e_head, player_snake[0], all_obstacles)
                        
                        if next_step: # 如果找得到路徑
                            e_snake.insert(0, next_step)
                        else:         # 如果被死路困住，就原地停滯一回合
                            e_snake.insert(0, e_head)
                            
                        e_snake.pop()

                # ====== 核心功能 2：能量護盾與無敵碰撞判定 ======
                collision_detected = False
                
                # 1. 撞到自己
                if new_head in player_snake[1:]:
                    collision_detected = True
                # 2. 撞到敵人
                for e_snake in enemy_snakes:
                    if new_head in e_snake or any(seg in e_snake for seg in player_snake):
                        collision_detected = True

                if collision_detected:
                    if invincible_timer > 0:
                        pass # 正在無敵狀態，免疫傷害穿透過去！
                    elif has_shield:
                        has_shield = False
                        invincible_timer = 90 # 消耗護盾，獲得 90 偵 (約1.5秒) 無敵時間
                    else:
                        state = STATE_GAMEOVER
                
                last_move_time = current_time

            # --- 畫面繪製 ---
            if score < target_score:
                pygame.draw.rect(screen, YELLOW, (food[0]*GRID_SIZE, food[1]*GRID_SIZE, GRID_SIZE, GRID_SIZE))
            elif exit_pos:
                pygame.draw.rect(screen, CYAN, (exit_pos[0]*GRID_SIZE, exit_pos[1]*GRID_SIZE, GRID_SIZE, GRID_SIZE))
            
            # 繪製護盾道具 (如果還沒被吃掉)
            if shield_item != [-1, -1]:
                pygame.draw.rect(screen, BLUE, (shield_item[0]*GRID_SIZE, shield_item[1]*GRID_SIZE, GRID_SIZE, GRID_SIZE), border_radius=5)
                
            for e_snake in enemy_snakes:
                for i, pos in enumerate(e_snake):
                    color = LIGHT_RED if i == 0 else RED
                    pygame.draw.rect(screen, color, (pos[0]*GRID_SIZE, pos[1]*GRID_SIZE, GRID_SIZE-1, GRID_SIZE-1))

            for i, pos in enumerate(player_snake):
                # 如果處於護盾啟動的無敵狀態，蛇頭會閃爍水藍色！
                if invincible_timer > 0 and (invincible_timer // 5) % 2 == 0:
                    color = SHIELD_AURA
                else:
                    color = LIGHT_GREEN if i == 0 else GREEN
                pygame.draw.rect(screen, color, (pos[0]*GRID_SIZE, pos[1]*GRID_SIZE, GRID_SIZE-1, GRID_SIZE-1))

            # UI 文字狀態
            shield_text = " [SHIELD READY]" if has_shield else ""
            status_text = font_small.render(f"Keys: {score}/{target_score}{shield_text}", True, WHITE)
            screen.blit(status_text, (10, 10))

        elif state == STATE_GAMEOVER or state == STATE_WIN:
            end_text = font_large.render("GAME OVER" if state == STATE_GAMEOVER else "SUCCESS!", True, LIGHT_RED if state == STATE_GAMEOVER else CYAN)
            screen.blit(end_text, (WIDTH//2 - end_text.get_width()//2, HEIGHT//2 - 80))
            score_text = font_small.render(f"Final Keys Collected: {score}/{target_score}", True, WHITE)
            screen.blit(score_text, (WIDTH//2 - score_text.get_width()//2, HEIGHT//2 - 10))

            restart_rect, restart_hovered = draw_button("RESTART", WIDTH//2 - 100, HEIGHT//2 + 30, 200, 50, mouse_pos)
            if mouse_clicked and restart_hovered:
                player_snake, player_dir, enemy_snakes, food, shield_item, score, exit_pos, enemy_move_counter, has_shield, invincible_timer, dash_speed_frames = init_game_vars()
                state = STATE_PLAYING

            exit_rect, exit_hovered = draw_button("EXIT", WIDTH//2 - 100, HEIGHT//2 + 90, 200, 50, mouse_pos)
            if mouse_clicked and exit_hovered:
                pygame.quit()
                sys.exit()

        pygame.display.flip()
        clock.tick(60)

if __name__ == "__main__":
    main()
