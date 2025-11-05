# Master-Chess-game
import React, { useState, useEffect, useCallback, useMemo } from 'react';

// --- Core Game Logic and Utilities (Reusable) ---

const BOARD_SIZE = 8;
const PIECES = {
    'R': '♜', 'N': '♞', 'B': '♝', 'Q': '♛', 'K': '♚', 'P': '♟', // Black (Uppercase in array)
    'r': '♖', 'n': '♘', 'b': '♗', 'q': '♕', 'k': '♔', 'p': '♙'  // White (Lowercase in array)
};

// Piece values for the AI evaluation function
const PIECE_VALUES = {
    'P': 10, 'N': 30, 'B': 30, 'R': 50, 'Q': 90, 'K': 900,
    'p': 10, 'n': 30, 'b': 30, 'r': 50, 'q': 90, 'k': 900
};

const INITIAL_BOARD = [
    ['R', 'N', 'B', 'Q', 'K', 'B', 'N', 'R'],
    ['P', 'P', 'P', 'P', 'P', 'P', 'P', 'P'],
    ['', '', '', '', '', '', '', ''],
    ['', '', '', '', '', '', '', ''],
    ['', '', '', '', '', '', '', ''],
    ['', '', '', '', '', '', '', ''],
    ['p', 'p', 'p', 'p', 'p', 'p', 'p', 'p'],
    ['r', 'n', 'b', 'q', 'k', 'b', 'n', 'r']
];

/** Converts algebraic notation (e.g., 'A1') to array coordinates [row, col]. */
const idToCoords = (id) => {
    const file = id.charAt(0).toUpperCase();
    const rank = parseInt(id.charAt(1));
    const col = file.charCodeAt(0) - 'A'.charCodeAt(0);
    const row = BOARD_SIZE - rank;
    return [row, col];
};

/** Converts array coordinates [row, col] to algebraic notation (e.g., 'A1'). */
const coordsToId = (row, col) => {
    const file = String.fromCharCode('A'.charCodeAt(0) + col);
    const rank = BOARD_SIZE - row;
    return `${file}${rank}`;
};

/** Gets the piece color based on case. */
const getPieceColor = (piece) => {
    if (!piece) return null;
    return piece === piece.toLowerCase() ? 'white' : 'black';
};

/** Checks if a move is blocked by any piece in between. (For Rooks, Bishops, Queens) */
const isPathBlocked = (r1, c1, r2, c2, board) => {
    const dr = Math.sign(r2 - r1);
    const dc = Math.sign(c2 - c1);
    let r = r1 + dr;
    let c = c1 + dc;
    while (r !== r2 || c !== c2) {
        if (board[r][c]) {
            return true;
        }
        r += dr;
        c += dc;
    }
    return false;
};

/**
 * Generates all valid basic moves for a piece (simplified, no check validation).
 * @returns {string[]} - Array of valid target cell IDs.
 */
const getValidMoves = (r, c, board) => {
    const piece = board[r][c];
    if (!piece) return [];

    const pieceType = piece.toUpperCase();
    const color = getPieceColor(piece);
    const potentialMoves = [];

    // Helper for sliding pieces (Rook, Bishop, Queen)
    const addSlidingMoves = (directions) => {
        for (const [dr, dc] of directions) {
            let nextR = r + dr;
            let nextC = c + dc;
            while (nextR >= 0 && nextR < BOARD_SIZE && nextC >= 0 && nextC < BOARD_SIZE) {
                const targetPiece = board[nextR][nextC];
                if (targetPiece) {
                    if (getPieceColor(targetPiece) !== color) {
                        potentialMoves.push(coordsToId(nextR, nextC));
                    }
                    break;
                }
                potentialMoves.push(coordsToId(nextR, nextC));
                nextR += dr;
                nextC += dc;
            }
        }
    };

    // Helper for single-step/jumping pieces (King, Knight)
    const addStepMoves = (deltas) => {
        for (const [dr, dc] of deltas) {
            const nextR = r + dr;
            const nextC = c + dc;
            if (nextR >= 0 && nextR < BOARD_SIZE && nextC >= 0 && nextC < BOARD_SIZE) {
                const targetPiece = board[nextR][nextC];
                if (!targetPiece || getPieceColor(targetPiece) !== color) {
                    potentialMoves.push(coordsToId(nextR, nextC));
                }
            }
        }
    };

    switch (pieceType) {
        case 'P': // Pawn
            const direction = (color === 'white') ? -1 : 1;
            const startRank = (color === 'white') ? 6 : 1;
            let moveR = r + direction;
            if (moveR >= 0 && moveR < BOARD_SIZE && !board[moveR][c]) {
                potentialMoves.push(coordsToId(moveR, c));
                if (r === startRank) {
                    let doubleR = r + 2 * direction;
                    if (!board[doubleR][c]) {
                        potentialMoves.push(coordsToId(doubleR, c));
                    }
                }
            }
            for (const dc of [-1, 1]) {
                const capR = r + direction;
                const capC = c + dc;
                if (capR >= 0 && capR < BOARD_SIZE && capC >= 0 && capC < BOARD_SIZE) {
                    const targetPiece = board[capR][capC];
                    if (targetPiece && getPieceColor(targetPiece) !== color) {
                        potentialMoves.push(coordsToId(capR, capC));
                    }
                }
            }
            break;
        case 'R': addSlidingMoves([[0, 1], [0, -1], [1, 0], [-1, 0]]); break;
        case 'B': addSlidingMoves([[1, 1], [1, -1], [-1, 1], [-1, -1]]); break;
        case 'Q': addSlidingMoves([[0, 1], [0, -1], [1, 0], [-1, 0], [1, 1], [1, -1], [-1, 1], [-1, -1]]); break;
        case 'K': addStepMoves([[0, 1], [0, -1], [1, 0], [-1, 0], [1, 1], [1, -1], [-1, 1], [-1, -1]]); break;
        case 'N': addStepMoves([[2, 1], [2, -1], [-2, 1], [-2, -1], [1, 2], [1, -2], [-1, 2], [-1, -2]]); break;
    }

    // Final path blocking check
    return potentialMoves.filter(targetId => {
        const [targetR, targetC] = idToCoords(targetId);
        const pieceType = piece.toUpperCase();
        if (['P', 'N', 'K'].includes(pieceType)) return true;
        return !isPathBlocked(r, c, targetR, targetC, board);
    });
};

/** Finds all pieces of a given color and their moves. */
const getAllMoves = (color, board) => {
    const allMoves = [];
    for (let r = 0; r < BOARD_SIZE; r++) {
        for (let c = 0; c < BOARD_SIZE; c++) {
            const piece = board[r][c];
            if (getPieceColor(piece) === color) {
                const moves = getValidMoves(r, c, board);
                if (moves.length > 0) {
                    allMoves.push({
                        from: coordsToId(r, c),
                        moves: moves
                    });
                }
            }
        }
    }
    return allMoves;
};

// --- AI Logic (MiniMax Implementation) ---

/**
 * Function to simulate a move on a board copy.
 * @returns {Array<Array<string>>} The new board state.
 */
const simulateMove = (board, fromId, toId) => {
    const [fromR, fromC] = idToCoords(fromId);
    const [toR, toC] = idToCoords(toId);

    const newBoard = board.map(row => [...row]);
    const movedPiece = newBoard[fromR][fromC];

    // 1. Execute Move
    newBoard[toR][toC] = movedPiece;
    newBoard[fromR][fromC] = '';

    // 2. Pawn Promotion (Auto-Queen)
    if (movedPiece.toUpperCase() === 'P') {
        if (getPieceColor(movedPiece) === 'white' && toR === 0) {
            newBoard[toR][toC] = 'q';
        } else if (getPieceColor(movedPiece) === 'black' && toR === 7) {
            newBoard[toR][toC] = 'Q';
        }
    }

    return newBoard;
};

/**
 * Basic material evaluation function. Positive score favors Black (AI).
 */
const evaluateBoard = (board) => {
    let score = 0;
    for (let r = 0; r < BOARD_SIZE; r++) {
        for (let c = 0; c < BOARD_SIZE; c++) {
            const piece = board[r][c];
            if (piece) {
                const value = PIECE_VALUES[piece.toUpperCase()] || 0;
                if (getPieceColor(piece) === 'black') {
                    score += value; // Black (Maximizer)
                } else {
                    score -= value; // White (Minimizer)
                }
            }
        }
    }
    return score;
};

/**
 * Recursive MiniMax function (simplified without Alpha-Beta pruning).
 * @param {Array<Array<string>>} board - The current board state.
 * @param {number} depth - How many more moves deep to search.
 * @param {boolean} isMaximizingPlayer - True if it's the AI's turn (Black).
 * @returns {number} The calculated score.
 */
const minimax = (board, depth, isMaximizingPlayer) => {
    if (depth === 0) {
        return evaluateBoard(board);
    }

    const color = isMaximizingPlayer ? 'black' : 'white';
    const possibleMoves = getAllMoves(color, board);

    if (possibleMoves.length === 0) {
        // Simple endgame check: If no moves, consider it a poor position
        return isMaximizingPlayer ? -9999 : 9999;
    }

    if (isMaximizingPlayer) {
        let maxEval = -Infinity;
        for (const { from, moves } of possibleMoves) {
            for (const to of moves) {
                const newBoard = simulateMove(board, from, to);
                const evaluation = minimax(newBoard, depth - 1, false); // Switch to minimizing player
                maxEval = Math.max(maxEval, evaluation);
            }
        }
        return maxEval;
    } else { // Minimizing Player (White)
        let minEval = Infinity;
        for (const { from, moves } of possibleMoves) {
            for (const to of moves) {
                const newBoard = simulateMove(board, from, to);
                const evaluation = minimax(newBoard, depth - 1, true); // Switch to maximizing player
                minEval = Math.min(minEval, evaluation);
            }
        }
        return minEval;
    }
};


/** Selects the best move using MiniMax based on difficulty. */
const getAIMove = (board, difficulty) => {
    // Set depth based on difficulty
    // INTERMEDIATE DIFFICULTY: Uses 2-ply search (better decision making)
    const MAX_DEPTH = (difficulty === 'easy') ? 1 : (difficulty === 'medium' ? 2 : 2); 

    const color = 'black';
    const possibleMoves = getAllMoves(color, board);

    if (possibleMoves.length === 0) return null;

    let bestMove = null;
    let maxEval = -Infinity;

    // The AI is the Maximizing Player (wants the highest score)
    for (const { from: fromId, moves } of possibleMoves) {
        for (const toId of moves) {
            // Simulate the move
            const newBoard = simulateMove(board, fromId, toId);

            // Calculate the evaluation after this move (looking 'MAX_DEPTH' ply deep)
            // We call minimax with isMaximizingPlayer=false because after the AI's move,
            // it will be the White (minimizing) player's turn to respond.
            const evaluation = minimax(newBoard, MAX_DEPTH - 1, false);

            if (evaluation > maxEval) {
                maxEval = evaluation;
                bestMove = { fromId, toId };
            }
        }
    }

    // Fallback: If all moves have the same score, pick one randomly (to avoid deterministic play)
    if (bestMove === null && possibleMoves.length > 0) {
        const randomPieceMove = possibleMoves[Math.floor(Math.random() * possibleMoves.length)];
        const fromId = randomPieceMove.from;
        const toId = randomPieceMove.moves[Math.floor(Math.random() * randomPieceMove.moves.length)];
        bestMove = { fromId, toId };
    }

    return bestMove;
};

// --- Storage for Wins/Ranks (Uses localStorage for simplicity) ---

const getRankData = () => {
    try {
        const data = localStorage.getItem('chessRanks');
        return data ? JSON.parse(data) : { wins: 0, losses: 0 };
    } catch (e) {
        console.error("Error loading rank data:", e);
        return { wins: 0, losses: 0 };
    }
};

const updateRankData = (result) => {
    const data = getRankData();
    if (result === 'win') {
        data.wins += 1;
    } else if (result === 'loss') {
        data.losses += 1;
    }
    localStorage.setItem('chessRanks', JSON.stringify(data));
};

// --- Reusable Components ---

/** Component to display the game board. */
const ChessBoard = ({ board, onCellClick, selectedCellId, validMoves, flipped }) => {
    const cells = [];
    const rows = flipped ? [7, 6, 5, 4, 3, 2, 1, 0] : [0, 1, 2, 3, 4, 5, 6, 7];
    const cols = flipped ? [7, 6, 5, 4, 3, 2, 1, 0] : [0, 1, 2, 3, 4, 5, 6, 7];

    rows.forEach(r => {
        cols.forEach(c => {
            const id = coordsToId(r, c);
            const pieceKey = board[r][c];
            const pieceSymbol = PIECES[pieceKey] || '';
            const isWhiteSquare = (r + c) % 2 === 0;

            let cellClasses = `cell flex items-center justify-center text-5xl cursor-pointer select-none transition-all duration-100 ${
                isWhiteSquare ? 'bg-gray-200' : 'bg-gray-500'
            }`;

            if (pieceKey) {
                cellClasses += pieceKey === pieceKey.toLowerCase() ? ' text-white drop-shadow-lg' : ' text-gray-800 drop-shadow-lg';
            }

            if (id === selectedCellId) {
                cellClasses = cellClasses.replace(/bg-gray-200|bg-gray-500/, 'bg-emerald-500');
                cellClasses += ' ring-4 ring-emerald-700 scale-[1.05] shadow-lg';
            }

            if (validMoves.includes(id)) {
                const isCapture = pieceKey !== '';
                // Highlight valid moves/captures
                cellClasses = cellClasses.replace(/bg-gray-200|bg-gray-500/, isCapture ? 'bg-red-300' : 'bg-yellow-300');
                if (isCapture) cellClasses += ' ring-4 ring-red-600';
                else cellClasses += ' ring-4 ring-yellow-400';
            }

            cells.push(
                <div key={id} id={id} className={cellClasses} onClick={() => onCellClick(id)}>
                    {pieceSymbol}
                </div>
            );
        });
    });

    return (
        <div
            id="chess-board"
            className="grid grid-cols-8 grid-rows-8 w-full max-w-lg aspect-square border-8 border-gray-700 shadow-2xl rounded-lg overflow-hidden"
            style={{ fontSize: 'clamp(30px, 8vw, 60px)' }}
        >
            {cells}
        </div>
    );
};

/** Component to display captured pieces. */
const CapturedPieces = ({ capturedPieces }) => {
    const whitePieces = capturedPieces.filter(p => getPieceColor(p) === 'white');
    const blackPieces = capturedPieces.filter(p => getPieceColor(p) === 'black');

    return (
        <div className="flex flex-col gap-4 p-4 bg-gray-700 rounded-lg shadow-inner w-full max-w-lg">
            <div className="flex flex-col">
                <p className="text-sm font-bold text-gray-300">White's Captures (Black Coins)</p>
                <div className="text-4xl text-gray-800 flex flex-wrap min-h-10">
                    {blackPieces.map((p, i) => (
                        <span key={i} className="mx-1">{PIECES[p]}</span>
                    ))}
                </div>
            </div>
            <div className="flex flex-col">
                <p className="text-sm font-bold text-gray-300">Black's Captures (White Coins)</p>
                <div className="text-4xl text-white flex flex-wrap min-h-10">
                    {whitePieces.map((p, i) => (
                        <span key={i} className="mx-1">{PIECES[p]}</span>
                    ))}
                </div>
            </div>
        </div>
    );
};

// --- Game Container (State and Actions) ---

/** Container for the actual game play, reusable for self-play and AI. */
const GameContainer = ({ gameMode = 'self', aiDifficulty = null, onGameOver }) => {
    const [board, setBoard] = useState(INITIAL_BOARD.map(row => [...row]));
    const [turn, setTurn] = useState('white');
    const [selectedId, setSelectedId] = useState(null);
    const [validMoves, setValidMoves] = useState([]);
    const [captured, setCaptured] = useState([]);
    const [gameOver, setGameOver] = useState(null); // 'white' | 'black' | 'draw' | null
    const [message, setMessage] = useState('');

    const isAIGame = gameMode === 'ai';

    // --- Core Move Function ---
    const performMove = useCallback((fromId, toId, currentBoard, currentCaptured) => {
        const [fromR, fromC] = idToCoords(fromId);
        const [toR, toC] = idToCoords(toId);

        const newBoard = currentBoard.map(row => [...row]);
        const newCaptured = [...currentCaptured];

        const movedPiece = newBoard[fromR][fromC];
        const capturedPiece = newBoard[toR][toC];

        if (capturedPiece) {
            newCaptured.push(capturedPiece);
        }

        // 1. Execute Move
        newBoard[toR][toC] = movedPiece;
        newBoard[fromR][fromC] = '';

        // 2. Pawn Promotion (Auto-Queen)
        let promoted = false;
        if (movedPiece.toUpperCase() === 'P') {
            if (getPieceColor(movedPiece) === 'white' && toR === 0) {
                newBoard[toR][toC] = 'q';
                promoted = true;
            } else if (getPieceColor(movedPiece) === 'black' && toR === 7) {
                newBoard[toR][toC] = 'Q';
                promoted = true;
            }
        }

        // 3. Check Win Condition (King Captured) - Simplified Checkmate
        let isGameOver = gameOver;
        if (capturedPiece && capturedPiece.toUpperCase() === 'K') {
            const winner = getPieceColor(movedPiece);
            isGameOver = winner;
            setGameOver(winner);
            setMessage(`Game Over! ${winner.toUpperCase()} wins by King Capture!`);
            onGameOver(winner);
        }

        // 4. Update State
        setBoard(newBoard);
        setCaptured(newCaptured);
        if (!isGameOver) {
            setTurn(prev => prev === 'white' ? 'black' : 'white');
        }
        setSelectedId(null);
        setValidMoves([]);
        setMessage(promoted ? `${getPieceColor(movedPiece).toUpperCase()} Pawn Promoted!` : '');

    }, [onGameOver, gameOver]);

    // --- User Click Handler ---
    const handleCellClick = (id) => {
        if (gameOver) return;

        // Prevent human input if it's the AI's turn
        if (isAIGame && turn === 'black') {
            setMessage("The computer is thinking...");
            return;
        }

        const [r, c] = idToCoords(id);
        const piece = board[r][c];
        const pieceColor = getPieceColor(piece);

        if (selectedId === null) {
            // Select a piece
            if (piece && pieceColor === turn) {
                setSelectedId(id);
                setValidMoves(getValidMoves(r, c, board));
            } else if (piece) {
                setMessage("That's not your piece!");
            }
        } else if (selectedId === id) {
            // Deselect
            setSelectedId(null);
            setValidMoves([]);
        } else if (validMoves.includes(id)) {
            // Execute Move
            performMove(selectedId, id, board, captured);
        } else if (piece && pieceColor === turn) {
            // Re-select another piece
            setSelectedId(id);
            setValidMoves(getValidMoves(r, c, board));
        } else {
            // Invalid move
            setMessage("Invalid move or target!");
        }
    };

    // --- AI Turn Logic (Effect) ---
    useEffect(() => {
        if (isAIGame && turn === 'black' && !gameOver) {
            setMessage("Computer's Turn...");

            // Delay for UX (make it look like the AI is thinking)
            const aiTimer = setTimeout(() => {
                const move = getAIMove(board, aiDifficulty);

                if (move) {
                    performMove(move.fromId, move.toId, board, captured);
                } else {
                    // No legal moves found - simplified Stalemate/Checkmate handling
                    setGameOver('draw');
                    setMessage("Stalemate! (No legal moves for the computer)");
                    onGameOver('draw');
                }
            }, 500);

            return () => clearTimeout(aiTimer);
        }
    }, [turn, isAIGame, gameOver, board, captured, aiDifficulty, performMove, onGameOver]);

    const handleRestart = () => {
        setBoard(INITIAL_BOARD.map(row => [...row]));
        setTurn('white');
        setSelectedId(null);
        setValidMoves([]);
        setCaptured([]);
        setGameOver(null);
        setMessage('');
    };

    return (
        <div className="flex flex-col items-center gap-6 p-4 md:flex-row md:items-start md:justify-center md:gap-10 w-full max-w-6xl mx-auto">
            <div className="flex flex-col items-center gap-4 w-full md:max-w-md order-2 md:order-1">
                <h2 className={`text-2xl font-bold p-3 rounded-xl shadow-lg w-full text-center ${
                    gameOver ? 'bg-red-600 text-white' : (turn === 'white' ? 'bg-white text-gray-800' : 'bg-gray-800 text-white')
                }`}>
                    {gameOver ? `${gameOver.toUpperCase()} WINS!` : `${turn.toUpperCase()}'s Turn`}
                </h2>
                {message && (
                    <p className="text-md font-semibold text-yellow-400">{message}</p>
                )}

                <CapturedPieces capturedPieces={captured} />

                <div className="flex gap-4 mt-2">
                    <button
                        onClick={handleRestart}
                        className="px-6 py-2 bg-indigo-600 text-white rounded-lg shadow-md hover:bg-indigo-700 transition duration-200"
                    >
                        Restart Game
                    </button>
                </div>
            </div>

            <div className="order-1 md:order-2">
                <ChessBoard
                    board={board}
                    onCellClick={handleCellClick}
                    selectedCellId={selectedId}
                    validMoves={validMoves}
                    // Flip board for white player in self-play mode (aesthetic choice)
                    flipped={gameMode === 'self' && turn === 'white'}
                />
            </div>
        </div>
    );
};

// --- Page Views ---

/** Main Menu and Navigation */
const HomeView = ({ setPage, rankData }) => {
    const level = useMemo(() => {
        if (rankData.wins >= 20) return "International Master";
        if (rankData.wins >= 10) return "National Champion";
        if (rankData.wins >= 5) return "State Level";
        return "Local Club Player";
    }, [rankData.wins]);

    return (
        <div className="flex flex-col items-center p-6 bg-gray-800 min-h-[calc(100vh-64px)] rounded-lg shadow-2xl">
            <h1 className="text-4xl font-extrabold text-white mb-8">Chess Training Center</h1>

            <div className="w-full max-w-sm p-6 bg-gray-700 rounded-xl shadow-lg mb-8 text-center">
                <h3 className="text-xl font-bold text-yellow-400">Your Current Rank</h3>
                <p className="text-3xl font-mono text-white mt-2">{level}</p>
                <p className="text-sm text-gray-300 mt-2">Wins: {rankData.wins} | Losses: {rankData.losses}</p>
            </div>

            <div className="flex flex-col gap-4 w-full max-w-sm">
                <button onClick={() => setPage('AI_SELECT')} className="py-4 bg-green-600 text-white text-xl font-bold rounded-xl shadow-md hover:bg-green-700 transition duration-300">
                    Practice with AI (Computer)
                </button>
                <button onClick={() => setPage('SELF_PLAY')} className="py-4 bg-blue-600 text-white text-xl font-bold rounded-xl shadow-md hover:bg-blue-700 transition duration-300">
                    Self-Practice (2-Player)
                </button>
                <button onClick={() => setPage('COMPETITION_LEVELS')} className="py-4 bg-indigo-600 text-white text-xl font-bold rounded-xl shadow-md hover:bg-indigo-700 transition duration-300">
                    Competition Levels & Rewards
                </button>
            </div>
        </div>
    );
};

/** Two-Player Mode */
const SelfPlayView = ({ setPage }) => (
    <div className="p-4 flex flex-col items-center">
        <h2 className="text-3xl font-bold text-white mb-6">Self-Practice Mode (Human vs. Human)</h2>
        <GameContainer gameMode="self" onGameOver={() => {}} />
        <button onClick={() => setPage('HOME')} className="mt-8 px-6 py-2 bg-gray-600 text-white rounded-lg hover:bg-gray-700">
            &larr; Back to Main Menu
        </button>
    </div>
);

/** AI Selection Page */
const AISelectView = ({ setPage, setAiDifficulty }) => {
    const difficulties = [
        { level: 'Beginner', desc: '1-Ply MiniMax: Chooses best immediate material gain.', color: 'bg-green-500', difficulty: 'easy' },
        { level: 'Intermediate', desc: '2-Ply MiniMax: Considers your best counter-move.', color: 'bg-yellow-500', difficulty: 'medium' },
        { level: 'Advanced', desc: 'Currently 2-Ply (Ready for deeper search or Alpha-Beta)', color: 'bg-red-500', difficulty: 'hard' },
    ];

    const handleSelect = (difficulty) => {
        setAiDifficulty(difficulty);
        setPage('AI_PLAY');
    };

    return (
        <div className="flex flex-col items-center p-6 bg-gray-800 min-h-[calc(100vh-64px)] rounded-lg shadow-2xl">
            <h2 className="text-3xl font-bold text-white mb-8">Select AI Difficulty</h2>
            <div className="flex flex-col gap-6 w-full max-w-md">
                {difficulties.map(d => (
                    <div key={d.level}
                         className={`p-5 rounded-xl shadow-lg text-center cursor-pointer transform hover:scale-[1.02] transition duration-200 ${d.color}`}
                         onClick={() => handleSelect(d.difficulty)}>
                        <h3 className="text-2xl font-bold text-white">{d.level}</h3>
                        <p className="text-sm text-gray-100 mt-1">{d.desc}</p>
                    </div>
                ))}
            </div>
            <button onClick={() => setPage('HOME')} className="mt-10 px-6 py-2 bg-gray-600 text-white rounded-lg hover:bg-gray-700">
                &larr; Back to Main Menu
            </button>
        </div>
    );
};

/** AI Play Mode */
const AIPlayView = ({ setPage, aiDifficulty, updateRankData }) => {
    const handleGameOver = (winner) => {
        if (winner === 'white') {
            updateRankData('win');
        } else if (winner === 'black') {
            updateRankData('loss');
        }
    };

    return (
        <div className="p-4 flex flex-col items-center">
            <h2 className="text-3xl font-bold text-white mb-6">AI Practice: {aiDifficulty.toUpperCase()}</h2>
            <GameContainer
                gameMode="ai"
                aiDifficulty={aiDifficulty}
                onGameOver={handleGameOver}
            />
            <button onClick={() => setPage('AI_SELECT')} className="mt-8 px-6 py-2 bg-gray-600 text-white rounded-lg hover:bg-gray-700">
                &larr; Change Difficulty
            </button>
        </div>
    );
};

/** Competition Levels Page */
const CompetitionLevelsView = ({ setPage, rankData }) => {
    const levels = [
        { name: "Local Club Player", threshold: 0, reward: "Master basic openings and checkmate patterns." },
        { name: "State Level", threshold: 5, reward: "Access to advanced game analysis tools." },
        { name: "National Champion", threshold: 10, reward: "Exclusive Gold board skin and title." },
        { name: "International Master", threshold: 20, reward: "Permanently unlock all future features and skins." },
    ];

    const currentWins = rankData.wins;

    return (
        <div className="flex flex-col items-center p-6 bg-gray-800 min-h-[calc(100vh-64px)] rounded-lg shadow-2xl w-full max-w-3xl mx-auto">
            <h2 className="text-3xl font-bold text-white mb-8">Competition Levels & Rewards</h2>
            <div className="space-y-6 w-full">
                {levels.map((level, index) => {
                    const isUnlocked = currentWins >= level.threshold;
                    const nextThreshold = levels[index + 1] ? levels[index + 1].threshold : null;
                    const progress = nextThreshold ? Math.min(100, (currentWins / nextThreshold) * 100) : 100;

                    return (
                        <div key={level.name} className={`p-5 rounded-xl shadow-xl transition-all duration-300 ${isUnlocked ? 'bg-indigo-600' : 'bg-gray-700'}`}>
                            <div className="flex justify-between items-center mb-2">
                                <h3 className={`text-2xl font-extrabold ${isUnlocked ? 'text-yellow-300' : 'text-white'}`}>{level.name}</h3>
                                <p className={`text-sm font-bold ${isUnlocked ? 'text-white' : 'text-gray-300'}`}>
                                    {isUnlocked ? 'UNLOCKED' : `Requires ${level.threshold} Wins`}
                                </p>
                            </div>
                            <p className={`text-md ${isUnlocked ? 'text-gray-100' : 'text-gray-400'}`}>
                                **Reward:** {level.reward}
                            </p>

                            {isUnlocked && nextThreshold && (
                                <div className="mt-3">
                                    <p className="text-xs text-white mb-1">Progress to Next Rank ({nextThreshold} Wins):</p>
                                    <div className="w-full bg-gray-500 rounded-full h-2.5">
                                        <div
                                            className="bg-yellow-400 h-2.5 rounded-full transition-all duration-500"
                                            style={{ width: `${progress}%` }}
                                        ></div>
                                    </div>
                                </div>
                            )}
                        </div>
                    );
                })}
            </div>
            <button onClick={() => setPage('HOME')} className="mt-10 px-6 py-2 bg-gray-600 text-white rounded-lg hover:bg-gray-700">
                &larr; Back to Main Menu
            </button>
        </div>
    );
};


// --- Main Application Component ---
const App = () => {
    // Simple state-based routing
    const [currentPage, setCurrentPage] = useState('HOME');
    // Defaulting to 'easy' which now uses 1-ply MiniMax
    const [aiDifficulty, setAiDifficulty] = useState('easy');
    const [rankData, setRankData] = useState(getRankData);

    // Function to update ranks and refresh state
    const handleUpdateRank = useCallback((result) => {
        updateRankData(result);
        setRankData(getRankData()); // Re-fetch from localStorage
    }, []);

    // Header for navigation
    const Header = () => (
        <header className="bg-gray-900 shadow-md p-4 flex justify-between items-center sticky top-0 z-10">
            <div className="text-white text-2xl font-bold cursor-pointer" onClick={() => setCurrentPage('HOME')}>
                ChessDev
            </div>
            <button
                onClick={() => setCurrentPage('HOME')}
                className="px-4 py-2 bg-gray-700 text-white rounded-lg text-sm font-semibold hover:bg-gray-600 transition"
            >
                Menu
            </button>
        </header>
    );

    const renderPage = () => {
        switch (currentPage) {
            case 'HOME':
                return <HomeView setPage={setCurrentPage} rankData={rankData} />;
            case 'SELF_PLAY':
                return <SelfPlayView setPage={setCurrentPage} />;
            case 'AI_SELECT':
                return <AISelectView setPage={setCurrentPage} setAiDifficulty={setAiDifficulty} />;
            case 'AI_PLAY':
                return <AIPlayView setPage={setCurrentPage} aiDifficulty={aiDifficulty} updateRankData={handleUpdateRank} />;
            case 'COMPETITION_LEVELS':
                return <CompetitionLevelsView setPage={setCurrentPage} rankData={rankData} />;
            default:
                return <HomeView setPage={setCurrentPage} rankData={rankData} />;
        }
    };

    return (
        <div className="min-h-screen bg-gray-900 font-sans">
            <Header />
            <main className="pt-8 pb-12">
                {renderPage()}
            </main>
        </div>
    );
};

export default App;

