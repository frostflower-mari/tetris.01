extends Sprite2D

#grid dimensions
const GRID_HEIGHT: int = 20
const GRID_WIDTH: int = 10
const GRID_SCALE: int = 15
const SPR_BLOCK: Texture2D = preload("res://spr_block.png")
const DEFAULT_COLOUR: Color = Color(0.2, 0.2, 0.2, 1)

var grid: Array = [] #array which contains all the game's information

var block_update_timer: float = .1
var do_block_update: bool = true
var do_new_controllable_block: bool = true
var player_control: bool = false
var row_complete: bool = false

var controllable_block_x: int = 0
var controllable_block_y: int = 0
var lowest_available_false: int = 0
var completed_row: int = 0


func _ready() -> void:
	#define each y-array
	for y in range(GRID_HEIGHT):
		grid.append([])
	
	#define x-arrays and add appropriate coordinates
	for y in range(len(grid)):
		for x in range(GRID_WIDTH):
			grid[y].append([Vector2(GRID_SCALE*x, GRID_SCALE*y), false])
	#array should be indexed as grid[y][x][element] to get intended position
	
	#draw grid blocks in the background
	for y in range(len(grid)):
		for x in range(GRID_WIDTH):
			var spr := Sprite2D.new()
			spr.texture = SPR_BLOCK
			spr.global_position = grid[y][x][0]
			spr.scale = Vector2(.5, .5)
			spr.self_modulate = DEFAULT_COLOUR
			add_child(spr)
			grid[y][x].append(spr)
	
	print(grid)

func _process(delta: float) -> void:
	#player input
	#go left only if the block to the left is empty and not a wall
	if Input.is_action_just_pressed("left") and controllable_block_x > 0 and !grid[controllable_block_y][controllable_block_x-1][1] and player_control:
		grid[controllable_block_y][controllable_block_x][1] = false
		controllable_block_x -= 1
		grid[controllable_block_y][controllable_block_x][1] = true
		_visual_update()
	
	#go right only if the block to the right is empty and not a wall
	if Input.is_action_just_pressed("right") and controllable_block_x < 9 and !grid[controllable_block_y][controllable_block_x+1][1] and player_control:
		grid[controllable_block_y][controllable_block_x][1] = false
		controllable_block_x += 1
		grid[controllable_block_y][controllable_block_x][1] = true
		_visual_update()
	
	#move down only if block below is empty and not the floor
	if Input.is_action_just_pressed("down") and controllable_block_y < 19 and !grid[controllable_block_y+1][controllable_block_x][1] and player_control:
		grid[controllable_block_y][controllable_block_x][1] = false
		controllable_block_y += 1
		grid[controllable_block_y][controllable_block_x][1] = true
		_visual_update()
	
	#drop to the lowest available space
	if Input.is_action_just_pressed("drop") and controllable_block_y < 19 and player_control:
		#locate lowest available 'false' space
		for y in range(len(grid)):
			if(grid[y][controllable_block_x][1] and y != controllable_block_y):
				lowest_available_false = y-1
				break #━┳━　━┳━
			elif(y == len(grid)-1):
				lowest_available_false = y
		
		grid[controllable_block_y][controllable_block_x][1] = false
		controllable_block_y = lowest_available_false
		grid[controllable_block_y][controllable_block_x][1] = true
		_visual_update()
	
	
	#wait for the duration of block_update_timer before running _block_update
	if do_block_update:
		do_block_update = false
		await get_tree().create_timer(block_update_timer).timeout
		_block_update()
		player_control = true
		do_block_update = true

#does all the tetris things
func _block_update() -> void:
	#print("doing _block_update")
	if do_new_controllable_block: #creates a new falling block that the player can control
		#if the position is already taken restart the game
		if grid[0][4][1]:
			_restart()
		else: #otherwise just make the new block
			grid[0][4][1] = true
			controllable_block_x = 4
			controllable_block_y = 0
			do_new_controllable_block = false
	#find position of controllable block and move it down one if possible
	elif controllable_block_y < 19 and !grid[controllable_block_y+1][controllable_block_x][1]: 
		grid[controllable_block_y][controllable_block_x][1] = false
		controllable_block_y += 1
		grid[controllable_block_y][controllable_block_x][1] = true
	#otherwise create new controllable block
	else:
		do_new_controllable_block = true
		_clear_complete_rows()
		_block_update()
	
	_visual_update()

#clear completed rows
func _clear_complete_rows():
	for y in range(GRID_HEIGHT - 1, -1, -1):  #index backwards to prevent fuckass indexing errors
		#detect where a row is complete
		var row_complete = true
		for x in range(GRID_WIDTH): #if any cell is empty, set row_complete to false and start over
			if !grid[y][x][1]:      #if an incomplete row is detected quit
				row_complete = false
				break #break out early if the row is incomplete
		
		if row_complete: #runs if the row is complete
			for x in range(GRID_WIDTH): #mark each cell in the complete row 'false'
				grid[y][x][1] = false
				
			for shift_y in range(y - 1, -1, -1): #all rows above complete row are shifted down 1
				for x in range(GRID_WIDTH):
					grid[shift_y + 1][x][1] = grid[shift_y][x][1]
			
			for x in range(GRID_WIDTH): #top row is cleared
				grid[0][x][1] = false
			
			y += 1 #redo loop for this row in case above row is also complete

#restart game
func _restart():
	#reset all grid values to false
	for y in range(len(grid)):
		for x in range(GRID_WIDTH):
			grid[y][x][1] = false

#update the visuals to match the grid
func _visual_update():
	for y in range(len(grid)):
			for x in range(GRID_WIDTH):
				if grid[y][x][1]: #if a cell is true colour it
					#print("true at (" + str(y) + ", " + str(x) + ")")
					grid[y][x][2].self_modulate = Color(.5, .5, .5, 1)
				else: #if a cell is false make it background colour
					grid[y][x][2].self_modulate = DEFAULT_COLOUR
