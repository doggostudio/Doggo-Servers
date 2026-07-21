# DoggoServers 1.0 - GameMaker Integration Guide

DoggoSRV is a lightweight Java TCP multiplayer server developed by Doggo Studios. It operates as a high-performance event relay service for multiplayer games built with GameMaker (GML).

---

## Overview Architecture

DoggoSRV does not handle positional grid physics or client-side game state calculations. Instead, it tracks player session states (PENDING, ACTIVE, SPECTATOR) and broadcasts state changes and player actions across all connected clients in real time.

---

## Protocol Specifications

* Protocol: Raw TCP Sockets
* Default Port: 5555
* Encoding: UTF-8 plain text
* Message Terminator: Line Feed (\n)
* Argument Delimiter: Pipe (|)

---

## Network Messages

### Client-to-Server Packets (GameMaker -> Server)

All requests sent from the game client follow the structure COMMAND|ARGUMENT1|ARGUMENT2.

| Command | Payload Example | Description |
| :--- | :--- | :--- |
| JOIN | JOIN|PlayerName | Requests initial registration on the server. Required upon connection. |
| MOVE | MOVE|UP | Broadcasts player movement (UP, DOWN, LEFT, RIGHT). |
| SHOOT | SHOOT|RIGHT | Broadcasts a shot action in a specific direction. |
| DIED | DIED | Notifies the server that the local player has been eliminated. |

---

### Server-to-Client Packets (Server -> GameMaker)

| Event | Payload Format | Description |
| :--- | :--- | :--- |
| ACCEPTED | ACCEPTED|<TEAM> | Registration accepted. Assigns player to RED or BLUE. |
| DECLINED | DECLINED | Registration denied (duplicate name or server full). |
| SPECTATOR | SPECTATOR | Confirms player transition to spectator status. |
| TEAM_CHANGED | TEAM_CHANGED|<TEAM> | Confirms team reassignment (RED or BLUE). |
| MATCH_STARTED | MATCH_STARTED|<RED_COUNT>|<BLUE_COUNT> | Broadcasts match start along with active player counts for each team. |
| PLAYER_MOVED | PLAYER_MOVED|<NAME>|<DIR> | Notifies that player <NAME> moved in <DIR> direction. |
| PLAYER_SHOT | PLAYER_SHOT|<NAME>|<DIR> | Notifies that player <NAME> fired in <DIR> direction. |
| PLAYER_ELIMINATED | PLAYER_ELIMINATED|<NAME> | Notifies that player <NAME> was eliminated. |
| GAME_ENDED | GAME_ENDED|<WINNING_TEAM> | Broadcasts match conclusion with the winning team (RED or BLUE). |

---

## Match Logic and Rules

1. Player Capacity: Up to 6 active players per match.
2. Team Balance: Matches require an even number of players distributed equally between RED and BLUE teams before starting.
3. Victory Condition: The win condition is team-based. When all active players of one team are eliminated, the opposing team wins.
4. Broadcast Declaration: The GAME_ENDED message carries the team name (e.g., GAME_ENDED|RED), ensuring all members of that team—surviving or eliminated—are recognized as winners.

---

## GameMaker (GML) Implementation

### 1. Client Object Creation (obj_client -> Create Event)

// Create TCP Raw Socket
socket = network_create_socket(network_socket_tcp);
var server_ip = "127.0.0.1";
var server_port = 5555;

// Connect to DoggoSRV
var connect_status = network_connect_raw(socket, server_ip, server_port);

if (connect_status >= 0) {
    show_debug_message("[DoggoSRV] Connected successfully.");
    
    // Helper function to transmit raw delimited lines
    function send_packet(str) {
        var payload = str + "\n";
        var buff = buffer_create(string_byte_length(payload), buffer_fixed, 1);
        buffer_write(buff, buffer_text, payload);
        network_send_raw(socket, buff, buffer_get_size(buff));
        buffer_delete(buff);
    }

    // Send registration request immediately
    my_name = "Player_" + string(irandom_range(1000, 9999));
    my_team = "";
    send_packet("JOIN|" + my_name);
} else {
    show_debug_message("[DoggoSRV] Connection failed.");
}

---

### 2. Processing Network Events (obj_client -> Async - Networking)

var net_type = async_load[? "type"];

if (net_type == network_type_data) {
    var rx_buffer = async_load[? "buffer"];
    var raw_string = buffer_read(rx_buffer, buffer_string);
    
    // Handle potential concatenated messages in a single buffer
    var packet_lines = string_split(raw_string, "\n");
    
    for (var i = 0; i < array_length(packet_lines); i++) {
        var packet = string_trim(packet_lines[i]);
        if (packet == "") continue;
        
        var tokens = string_split(packet, "|");
        var header = tokens[0];
        
        switch (header) {
            case "ACCEPTED":
                my_team = tokens[1];
                show_debug_message("[DoggoSRV] Joined team: " + my_team);
                break;
                
            case "DECLINED":
                show_debug_message("[DoggoSRV] Connection declined by host.");
                network_destroy(socket);
                break;
                
            case "SPECTATOR":
                show_debug_message("[DoggoSRV] Set to spectator mode.");
                break;
                
            case "TEAM_CHANGED":
                my_team = tokens[1];
                show_debug_message("[DoggoSRV] Switched to team: " + my_team);
                break;
                
            case "MATCH_STARTED":
                var red_count = real(tokens[1]);
                var blue_count = real(tokens[2]);
                show_debug_message("[DoggoSRV] Match started! RED: " + string(red_count) + " | BLUE: " + string(blue_count));
                break;
                
            case "PLAYER_MOVED":
                var target_player = tokens[1];
                var move_dir = tokens[2];
                
                with (obj_other_player) {
                    if (player_name == target_player) {
                        if (move_dir == "UP")    y -= 4;
                        if (move_dir == "DOWN")  y += 4;
                        if (move_dir == "LEFT")  x -= 4;
                        if (move_dir == "RIGHT") x += 4;
                    }
                }
                break;
                
            case "PLAYER_SHOT":
                var shooter_name = tokens[1];
                var shoot_dir = tokens[2];
                
                with (obj_other_player) {
                    if (player_name == shooter_name) {
                        // Execute shooting animation or instantiate projectile
                    }
                }
                break;
                
            case "PLAYER_ELIMINATED":
                var dead_player = tokens[1];
                
                with (obj_other_player) {
                    if (player_name == dead_player) {
                        instance_destroy();
                    }
                }
                break;
                
            case "GAME_ENDED":
                var winning_team = tokens[1];
                if (my_team == winning_team) {
                    show_debug_message("[DoggoSRV] Victory! Team " + winning_team + " won the match.");
                } else {
                    show_debug_message("[DoggoSRV] Defeat! Team " + winning_team + " won the match.");
                }
                break;
        }
    }
}

---

### 3. Handling Player Controls (obj_client -> Step Event)

// Send movement actions
if (keyboard_check_pressed(ord("W")) || keyboard_check_pressed(vk_up)) {
    send_packet("MOVE|UP");
}
if (keyboard_check_pressed(ord("S")) || keyboard_check_pressed(vk_down)) {
    send_packet("MOVE|DOWN");
}
if (keyboard_check_pressed(ord("A")) || keyboard_check_pressed(vk_left)) {
    send_packet("MOVE|LEFT");
}
if (keyboard_check_pressed(ord("D")) || keyboard_check_pressed(vk_right)) {
    send_packet("MOVE|RIGHT");
}

// Send combat action
if (keyboard_check_pressed(vk_space)) {
    send_packet("SHOOT|RIGHT");
}
