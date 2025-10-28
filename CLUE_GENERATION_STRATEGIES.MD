1. Post-Processing Code: This is often the most reliable method.
    - Let the LLM generate the clues within the JSON structure.
    - After receiving the response, parse the JSON in your code (e.g., Python).
    - Iterate through each clue string.
    - Count the words (len(clue.split())).
    - If a clue is outside the 15-20 range:
        - If a clue is too short or too long, make another targeted LLM call specifically asking it to rewrite only that specific clue to be 15-20 words while preserving its core meaning and type (informed/misinformed/fake). Example prompt: "Rewrite the following clue to be exactly 15-20 words long, keeping its original meaning: '[original clue text]'
        - Replace the non-compliant clue in your parsed JSON data with the corrected version.
    - Use the fully validated/corrected JSON data for your game.

