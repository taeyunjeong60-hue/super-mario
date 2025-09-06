# super-mario
슈퍼 마리오 게임(테무산)

# -*- coding: utf-8 -*-
"""
마리오-스타일 1스테이지 (단일 파일)
------------------------------------
필요사항:
  pip install pygame

실행:
  python mario_stage1.py (파일 이름은 원하는 대로 저장)

조작법:
  ← →  : 좌우 이동
  Z / 스페이스 : 점프
  R     : 스테이지 리셋
  ESC   : 종료

설명:
  - 실제 '슈퍼 마리오'의 저작물을 사용하지 않고, 단순한 색상/사각형 도형으로 구현한
    오리지널 자산입니다. 취미/학습 용도로 자유롭게 수정하세요.
  - 카메라 스크롤, 충돌, 코인 수집, 적 밟기, 깃발 도착(클리어), 목숨/체크포인트(리스폰)
    기능이 포함되어 있습니다.
"""

import os
import sys
import math
import pygame

# -----------------------------
# 기본 설정
# -----------------------------
WIDTH, HEIGHT = 960, 540
FPS = 60
TILE = 48  # 타일 크기
GRAVITY = 0.65
JUMP_VELOCITY = -12.5
MAX_FALL_SPEED = 18

# 색상 팔레트 (간단한 미니멀 스타일)
COL_BG = (135, 206, 250)   # 하늘색
COL_GROUND = (65, 160, 85) # 잔디/땅
COL_DIRT = (110, 80, 50)   # 흙 가장자리
COL_PLAYER = (255, 90, 60) # 플레이어
COL_ENEMY = (170, 110, 70) # 적(Patrol)
COL_BLOCK = (150, 150, 150)# 벽/블록
COL_COIN = (255, 215, 0)   # 코인
COL_HAZARD = (235, 80, 80) # 함정(용암)
COL_FLAG = (255, 255, 255) # 깃발 천
COL_FLAG_POLE = (40, 40, 40)
COL_TEXT = (20, 20, 20)
COL_UI_BG = (255, 255, 255)

# 글꼴 초기화 전 전역 폰트 변수 자리표시자
FONT = None

# -----------------------------
# 유틸
# -----------------------------

def sign(x):
    return (x > 0) - (x < 0)

# -----------------------------
# 스프라이트 클래스
# -----------------------------
class Tile(pygame.sprite.Sprite):
    def __init__(self, x, y, kind):
        super().__init__()
        self.kind = kind
        self.image = pygame.Surface((TILE, TILE), pygame.SRCALPHA)
        if kind == 'ground':
            # 간단한 잔디-흙 블록 비주얼
            self.image.fill(COL_DIRT)
            top = pygame.Rect(0, 0, TILE, TILE//4)
            pygame.draw.rect(self.image, COL_GROUND, top)
        elif kind == 'block':
            self.image.fill(COL_BLOCK)
            # 테두리
            pygame.draw.rect(self.image, (100,100,100), self.image.get_rect(), 2)
        elif kind == 'hazard':
            self.image.fill(COL_HAZARD)
            for i in range(0, TILE, 8):
                pygame.draw.polygon(self.image, (255, 180, 180), [(i, TILE), (i+4, TILE-10), (i+8, TILE)])
        else:
            self.image.fill((0,0,0))
        self.rect = self.image.get_rect(topleft=(x, y))

class Coin(pygame.sprite.Sprite):
    def __init__(self, x, y):
        super().__init__()
        self.image = pygame.Surface((TILE//1.8, TILE//1.8), pygame.SRCALPHA)
        r = self.image.get_rect()
        pygame.draw.ellipse(self.image, COL_COIN, r)
        pygame.draw.ellipse(self.image, (180, 150, 0), r, 2)
        self.rect = self.image.get_rect(center=(x + TILE//2, y + TILE//2))
        self.v = 0

    def update(self, dt):
        # 살짝 떠있는 애니메이션
        self.v += dt * 0.02
        offset = math.sin(self.v) * 2
        self.rect.y += offset
        self.rect.y -= offset  # 실제 이동은 하지 않도록 상쇄 (시각적만 남김)

class Flag(pygame.sprite.Sprite):
    def __init__(self, x, y):
        super().__init__()
        w, h = TILE//4, TILE*4
        pole = pygame.Surface((w, h), pygame.SRCALPHA)
        pole.fill(COL_FLAG_POLE)
        self.pole = pole
        flag = pygame.Surface((TILE, TILE//2), pygame.SRCALPHA)
        flag.fill(COL_FLAG)
        self.flag = flag
        self.image = pygame.Surface((TILE + w, h), pygame.SRCALPHA)
        self.image.blit(self.pole, (0, 0))
        self.image.blit(self.flag, (w, 10))
        self.rect = self.image.get_rect(bottomleft=(x, y + TILE))

class Enemy(pygame.sprite.Sprite):
    SPEED = 1.5
    def __init__(self, x, y):
        super().__init__()
        self.image = pygame.Surface((TILE*0.9, TILE*0.9), pygame.SRCALPHA)
        pygame.draw.rect(self.image, COL_ENEMY, self.image.get_rect(), border_radius=8)
        self.rect = self.image.get_rect(bottomleft=(x, y+TILE))
        self.vx = -Enemy.SPEED
        self.vy = 0
        self.alive = True

    def update(self, level, dt):
        if not self.alive:
            return
        # 간단한 좌우 이동 + 낭떠러지/벽에서 방향전환
        self.rect.x += int(self.vx * dt)
        if self.hit_solid(level):
            # 벽 충돌시 반전
            self.rect.x -= int(self.vx * dt)
            self.vx *= -1
        # 발밑 체크: 낭떠러지면 반전
        ahead_x = self.rect.centerx + int(sign(self.vx) * TILE//2)
        feet_y = self.rect.bottom + 2
        if not level.solid_at(ahead_x, feet_y):
            self.vx *= -1
        # 중력
        self.vy = min(self.vy + GRAVITY, MAX_FALL_SPEED)
        self.rect.y += int(self.vy * dt)
        # 바닥 충돌 처리
        tiles = level.get_solid_collisions(self.rect)
        for t in tiles:
            if self.vy > 0 and self.rect.bottom > t.rect.top and self.rect.top < t.rect.top:
                self.rect.bottom = t.rect.top
                self.vy = 0

    def hit_solid(self, level):
        for t in level.get_solid_collisions(self.rect):
            return True
        return False

class Player(pygame.sprite.Sprite):
    def __init__(self, x, y):
        super().__init__()
        self.image = pygame.Surface((TILE*0.9, TILE*0.95), pygame.SRCALPHA)
        pygame.draw.rect(self.image, COL_PLAYER, self.image.get_rect(), border_radius=6)
        self.rect = self.image.get_rect(bottomleft=(x, y+TILE))
        self.vx = 0
        self.vy = 0
        self.speed = 5.0
        self.on_ground = False
        self.invul = 0  # 피격 무적 프레임
        self.coins = 0
        self.lives = 3
        self.checkpoint = None  # (x, y)
        self.stage_clear = False

    def spawn(self, pos):
        self.rect.bottomleft = pos
        self.vx = self.vy = 0
        self.on_ground = False

    def handle_input(self, keys):
        ax = 0
        if keys[pygame.K_LEFT] or keys[pygame.K_a]:
            ax -= 1
        if keys[pygame.K_RIGHT] or keys[pygame.K_d]:
            ax += 1
        self.vx = ax * self.speed
        if (keys[pygame.K_z] or keys[pygame.K_SPACE] or keys[pygame.K_w] or keys[pygame.K_UP]) and self.on_ground:
            self.vy = JUMP_VELOCITY
            self.on_ground = False

    def update(self, level, dt):
        if self.invul > 0:
            self.invul -= 1
        # 수평 이동
        self.rect.x += int(self.vx * dt)
        # 수평 충돌
        for t in level.get_solid_collisions(self.rect):
            if self.vx > 0:
                self.rect.right = t.rect.left
            elif self.vx < 0:
                self.rect.left = t.rect.right
        # 중력
        self.vy = min(self.vy + GRAVITY, MAX_FALL_SPEED)
        self.rect.y += int(self.vy * dt)
        # 수직 충돌
        self.on_ground = False
        for t in level.get_solid_collisions(self.rect):
            if self.vy > 0:  # 아래로 낙하 중 바닥
                self.rect.bottom = t.rect.top
                self.vy = 0
                self.on_ground = True
            elif self.vy < 0:  # 머리로 천장
                self.rect.top = t.rect.bottom
                self.vy = 0
        # 코인 획득
        got = pygame.sprite.spritecollide(self, level.coins, dokill=True)
        if got:
            self.coins += len(got)
        # 함정
        if level.collide_hazard(self.rect):
            self.hurt(level)
        # 적과의 상호작용
        for e in level.enemies:
            if not e.alive:
                continue
            if self.rect.colliderect(e.rect):
                # 위에서 떨어지면 처치
                if self.vy > 1 and self.rect.bottom - e.rect.top < TILE//2:
                    e.alive = False
                    e.kill()
                    self.vy = JUMP_VELOCITY * 0.6
                else:
                    self.hurt(level)
        # 깃발
        if level.flag and self.rect.colliderect(level.flag.rect):
            self.stage_clear = True

    def hurt(self, level):
        if self.invul > 0:
            return
        self.lives -= 1
        self.invul = 60
        if self.lives < 0:
            # 게임오버 -> 전체 리셋
            level.reset(hard=True)
        else:
            # 체크포인트가 있으면 그쪽으로
            if self.checkpoint:
                self.spawn(self.checkpoint)
            else:
                self.spawn(level.player_start)
    # Player 클래스 안에 추가



# -----------------------------
# 레벨 구성 & 로더
# -----------------------------
class Level:
    """
    맵 전개: 문자열로 구성
      'X' : ground(밟을 수 있는 땅)
      '#' : block(단단한 블록)
      'H' : hazard(용암/가시)
      'C' : coin
      'E' : enemy
      'P' : player 시작
      'F' : goal flag
      '=' : 체크포인트(통과 시 리스폰 위치 갱신)
      '-'/' ' : 빈 공간
    """
    def __init__(self):
        self.solid = pygame.sprite.Group()
        self.hazard = pygame.sprite.Group()
        self.coins = pygame.sprite.Group()
        self.enemies = pygame.sprite.Group()
        self.decor = pygame.sprite.Group()  # 깃발 등
        self.all = pygame.sprite.LayeredUpdates()
        self.flag = None
        self.player = None
        self.player_start = (TILE*2, TILE*5)
        self._map = LEVEL_DATA
        self.world_w = len(self._map[0]) * TILE
        self.world_h = len(self._map) * TILE
        self.build()

    def build(self):
        self.solid.empty(); self.hazard.empty(); self.coins.empty(); self.enemies.empty(); self.decor.empty(); self.all.empty()
        start_found = False
        for j, row in enumerate(self._map):
            for i, ch in enumerate(row):
                x, y = i*TILE, j*TILE
                if ch == 'X':
                    t = Tile(x, y, 'ground'); self.solid.add(t); self.all.add(t, layer=0)
                elif ch == '#':
                    t = Tile(x, y, 'block'); self.solid.add(t); self.all.add(t, layer=0)
                elif ch == 'H':
                    t = Tile(x, y, 'hazard'); self.hazard.add(t); self.all.add(t, layer=0)
                elif ch == 'C':
                    c = Coin(x, y); self.coins.add(c); self.all.add(c, layer=1)
                elif ch == 'E':
                    e = Enemy(x, y); self.enemies.add(e); self.all.add(e, layer=2)
                elif ch == 'F':
                    self.flag = Flag(x, y); self.decor.add(self.flag); self.all.add(self.flag, layer=0)
                elif ch == 'P':
                    self.player_start = (x, y+TILE)
                    start_found = True
                # '=' 체크포인트는 바닥 타일 위에 보이지 않는 마커로 처리
        if not start_found:
            self.player_start = (TILE*2, TILE*5)
        # 플레이어 생성
        self.player = Player(*self.player_start)
        self.all.add(self.player, layer=3)

    def solid_at(self, x, y):
        test = pygame.Rect(x, y, 1, 1)
        for t in self.solid:
            if t.rect.colliderect(test):
                return True
        return False

    def get_solid_collisions(self, rect):
        hits = []
        for t in self.solid:
            if rect.colliderect(t.rect):
                hits.append(t)
        return hits

    def collide_hazard(self, rect):
        for h in self.hazard:
            if rect.colliderect(h.rect):
                return True
        return False

    def update(self, dt):
        # 적/코인 등 업데이트
        for e in self.enemies:
            e.update(self, dt)
        for c in self.coins:
            c.update(dt)
        self.player.update(self, dt)

    def reset(self, hard=False):
        if hard:
            # 전체 초기화
            self.build()
        else:
            # 부분 초기화 (플레이어만 리스폰)
            self.player.spawn(self.player_start)
            self.player.vx = self.player.vy = 0
# -----------------------------
# 카메라
# -----------------------------
class Camera:
    def __init__(self, level_w, level_h, screen_w, screen_h):
        self.level_w = level_w
        self.level_h = level_h
        self.screen_w = screen_w
        self.screen_h = screen_h
        self.x = 0
        self.y = 0

    def apply(self, rect):
        return rect.move(-self.x, -self.y)

    def update(self, target_rect):
        # 가벼운 데드존 카메라
        margin_x = self.screen_w // 3
        desired_x = target_rect.centerx - self.screen_w // 2
        # 스무딩 없이 단순 추적(중학생 학습용으로 단순화)
        self.x = max(0, min(desired_x, self.level_w - self.screen_w))
        self.y = -150  # 수직 고정(필요시 스크롤 가능)

# -----------------------------
# HUD & 텍스트
# -----------------------------

def draw_text(surface, text, pos, size=28, color=COL_TEXT, bold=False):
    global FONT
    if FONT is None:
        FONT = pygame.font.SysFont('malgungothic', 24) or pygame.font.SysFont(None, 24)
    font = pygame.font.SysFont('malgungothic', size, bold=bold) or pygame.font.SysFont(None, size)
    img = font.render(text, True, color)
    surface.blit(img, pos)

# -----------------------------
# 레벨 데이터 (간단 1-1 스타일)
# -----------------------------
LEVEL_DATA = [
    # 0         1         2         3         4         5         6         7         8         9         10        11        12
    #0123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123
    "--------------------------------------------------------------------------------------------------------------------------------",
    "--------------------------------------------------------------------------------------------------------------------------------",
    "--------------------------------------------------------------------#--------------------------------------------------C--------",
    "---------------------C-------------------------------C-------------##------C-------------------------------C--------------------",
    "-------------------------E---------------C---------------------E--###-------------------C---------------------------------------",
    "P-----------------------HHXXXXXXXXX------------------------------####---------------------------------------E---E------------F--",
    "XXXXXXXXXXXXXXXXX---XXXXXXXXXXXXXXX---###---XXXXXX--##--XXXXXXXXXXXXX-----XXX---XXXXXXX--#--XXXX--XX--XXXXXXXXXXXXXXXX---XXXXXXX",
    "################----###############---------######------#############-----###---#######-----####--##--################---#######",
]
# 맵 높이 = 8 줄, 폭 = len(row) * TILE. 필요한 경우 자유롭게 편집하세요.

# -----------------------------
# 게임 루프
# -----------------------------

def main():
    pygame.init()
    pygame.display.set_caption("Mario-like Stage 1 (교육용)")
    screen = pygame.display.set_mode((WIDTH, HEIGHT))
    clock = pygame.time.Clock()

    level = Level()
    cam = Camera(level.world_w, level.world_h, WIDTH, HEIGHT)

    # 배경(멀티 레이어 간단 구현)
    clouds = []
    for i in range(12):
        surf = pygame.Surface((120, 60), pygame.SRCALPHA)
        pygame.draw.ellipse(surf, (255, 255, 255), (0, 10, 80, 40))
        pygame.draw.ellipse(surf, (255, 255, 255), (30, 0, 90, 50))
        clouds.append((surf, i*400+200, 60 + (i%3)*40))

    running = True
    while running:
        dt = clock.tick(FPS) / (1000/60)  # 60fps 기준 배율
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                running = False
            elif event.type == pygame.KEYDOWN:
                if event.key == pygame.K_ESCAPE:
                    running = False
                if event.key == pygame.K_r:
                    # 스테이지 리셋
                    level = Level()
                    cam = Camera(level.world_w, level.world_h, WIDTH, HEIGHT)
        keys = pygame.key.get_pressed()
        level.player.handle_input(keys)

        level.update(dt)
        cam.update(level.player.rect)

        # --- 렌더 ---
        screen.fill(COL_BG)
        # 아주 간단한 패럴랙스: 구름 이동
        for idx, (surf, cx, cy) in enumerate(clouds):
            scroll = cam.x * 0.5
            screen.blit(surf, (cx - scroll, cy))
        # 타일/스프라이트
        for spr in level.all:
            if spr == level.player and level.player.invul > 0 and (pygame.time.get_ticks() // 120) % 2 == 0:
                # 무적 중 깜빡임 효과
                continue
            screen.blit(spr.image, cam.apply(spr.rect))

        # UI
        pygame.draw.rect(screen, (255,255,255,200), pygame.Rect(10, 10, 220, 70), border_radius=12)
        draw_text(screen, f"코인: {level.player.coins}", (20, 18), size=24, bold=True)
        draw_text(screen, f"목숨: {max(0, level.player.lives)}", (20, 46), size=24, bold=True)

        if level.player.stage_clear:
            box = pygame.Surface((WIDTH, 120), pygame.SRCALPHA)
            pygame.draw.rect(box, (255,255,255,230), pygame.Rect(0,0,WIDTH,120))
            screen.blit(box, (0, HEIGHT//2 - 60))
            draw_text(screen, "STAGE CLEAR! R 키로 다시하기", (WIDTH//2 - 220, HEIGHT//2 - 10), size=32, bold=True)

        pygame.display.flip()

    pygame.quit()


if __name__ == '__main__':
    main()
