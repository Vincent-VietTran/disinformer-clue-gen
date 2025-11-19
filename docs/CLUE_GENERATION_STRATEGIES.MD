1. Post-Processing Code: This is often the most reliable method.
    - Let the LLM generate the clues within the JSON structure.
    - After receiving the response, parse the JSON in your code (e.g., Python).
    - Iterate through each clue string.
    - Count the words (len(clue.split())).
    - If a clue is outside the 15-20 range:
        - If a clue is too short or too long, make another targeted LLM call specifically asking it to rewrite only that specific clue to be 15-20 words while preserving its core meaning and type (informed/misinformed/fake). Example prompt: "Rewrite the following clue to be exactly 15-20 words long, keeping its original meaning: '[original clue text]'
        - Could also mannually refine the unqualified clues to make it valid in terms of length
        - Replace the non-compliant clue in your parsed JSON data with the corrected version.
    - Use the fully validated/corrected JSON data for your game.
    - fix/consideration to avoid reaching RPM (request per minute) limit
        - Rewrite failed clue in batch instead of every single one
        


2. generate more clues, then filter out best
9 informed
2 fake
2 misinformed
1 extra


3. shorter prompt used to generate one clue at a time, put in a for loop, generate in order 9 informed clues, then 2 misinformed, then 2 fake, then 1 extra, at each iteration, if clue doesnt match length requirements, regenerate until match (using same prompt or another prompt to refine current under-qualified clue)