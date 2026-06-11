import json
import os
import random
import urllib.error
import urllib.request

import streamlit as st

GEMINI_API_URL = 'https://api.openai.com/v1/responses'

crime_scene = {
    'location': 'Victim’s study',
    'description': 'A quiet room with a desk, a broken window, and a spilled cup of coffee.',
}

victim = 'Unknown'
suspects = []
clues = []
contradictions = []
score = 0
found_contradictions = []


def get_api_key():
    return os.getenv('GEMINI_API_KEY')


def ask_gemini(prompt):
    api_key = get_api_key()
    if not api_key:
        raise EnvironmentError('Please set GEMINI_API_KEY in your environment.')

    headers = {
        'Authorization': f'Bearer {api_key}',
        'Content-Type': 'application/json',
    }
    request_data = json.dumps({
        'model': 'gemini-1.5',
        'input': prompt,
    }).encode('utf-8')

    request = urllib.request.Request(GEMINI_API_URL, data=request_data, headers=headers, method='POST')
    with urllib.request.urlopen(request, timeout=20) as response:
        return json.load(response)


def extract_text_from_response(response):
    if not isinstance(response, dict):
        return ''
    output = response.get('output')
    if isinstance(output, list) and output:
        first = output[0]
        if isinstance(first, dict):
            content = first.get('content')
            if isinstance(content, list):
                for item in content:
                    if isinstance(item, dict) and 'text' in item:
                        return item['text']
            if 'text' in first:
                return first['text']
        if isinstance(first, str):
            return first
    if isinstance(output, str):
        return output
    return ''


def parse_gemini_json(text):
    try:
        return json.loads(text)
    except json.JSONDecodeError:
        start = text.find('{')
        end = text.rfind('}') + 1
        if start != -1 and end != -1:
            return json.loads(text[start:end])
        raise


def generate_default_data():
    return {
        'victim': 'Dr. Harper',
        'suspects': [
            {'name': 'Avery', 'alibi': 'I was reading in the library.', 'motive': 'They were angry about a promotion.'},
            {'name': 'Blake', 'alibi': 'I was cooking dinner in the kitchen.', 'motive': 'They argued with the victim yesterday.'},
            {'name': 'Casey', 'alibi': 'I was fixing the broken study window.', 'motive': 'They needed money back from the victim.'},
        ],
        'clues': [
            {'text': 'A spilled cup of coffee on the desk.'},
            {'text': 'A broken window with glass on the floor.'},
            {'text': 'A note with the victim’s name and a time written in red ink.'},
        ],
        'contradictions': [
            {'suspect': 'Blake', 'clue': 'A spilled cup of coffee on the desk.'},
            {'suspect': 'Casey', 'clue': 'A broken window with glass on the floor.'},
            {'suspect': 'Avery', 'clue': 'A library book open to a chapter about night patrols.'},
        ],
    }


def normalize_gemini_data(data):
    victim_data = data.get('victim', 'Unknown')
    suspects_data = []
    raw_suspects = data.get('suspects', [])
    for item in raw_suspects:
        if isinstance(item, dict) and 'name' in item:
            suspects_data.append({
                'name': item['name'],
                'alibi': item.get('alibi', 'No alibi given.'),
                'motive': item.get('motive', 'No motive given.'),
            })
    clues_data = []
    raw_clues = data.get('clues', [])
    for item in raw_clues:
        if isinstance(item, str):
            clues_data.append({'text': item})
        elif isinstance(item, dict) and 'text' in item:
            clues_data.append({'text': item['text']})
    contradictions_data = []
    raw_contradictions = data.get('contradictions', [])
    for item in raw_contradictions:
        if isinstance(item, dict) and 'suspect' in item and 'clue' in item:
            contradictions_data.append({
                'suspect': item['suspect'],
                'clue': item['clue'],
            })
    return {
        'victim': victim_data,
        'suspects': suspects_data,
        'clues': clues_data,
        'contradictions': contradictions_data,
    }


def generate_game_data():
    prompt = (
        'Create a murder mystery scenario as valid JSON only. '
        'Return victim, suspects, clues, and contradictions. '
        'Use 3 suspects. Each suspect must include name, alibi, and motive. '
        'The clues list must contain clue text strings. '
        'The contradictions list must contain objects with suspect and clue. '
        'Do not add any extra explanation outside the JSON object.'
    )

    try:
        response = ask_gemini(prompt)
        text = extract_text_from_response(response)
        data = parse_gemini_json(text)
        normalized = normalize_gemini_data(data)

        if len(normalized['suspects']) != 3 or not normalized['clues'] or not normalized['contradictions']:
            return generate_default_data()
        return normalized
    except (EnvironmentError, urllib.error.URLError, ValueError, json.JSONDecodeError):
        return generate_default_data()


def initialize_game():
    game_data = generate_game_data()
    st.session_state.victim = game_data['victim']
    st.session_state.suspects = game_data['suspects']
    st.session_state.clues = game_data['clues']
    st.session_state.contradictions = game_data['contradictions']
    st.session_state.murderer_index = random.randrange(len(st.session_state.suspects))
    st.session_state.suspects[st.session_state.murderer_index]['guilty'] = True
    st.session_state.score = 0
    st.session_state.found_contradictions = []
    st.session_state.interrogations = 0
    st.session_state.max_interrogations = 3
    st.session_state.case_over = False
    st.session_state.result_message = 'Start your investigation by viewing clues and interrogating suspects.'
    st.session_state.interrogated = []


def find_contradiction(suspect):
    return next((item for item in st.session_state.contradictions if item['suspect'] == suspect['name']), None)


def interrogate_index(index):
    if st.session_state.case_over:
        return

    suspect = st.session_state.suspects[index]
    if index in st.session_state.interrogated:
        st.session_state.result_message = f'You have already interrogated {suspect["name"]}. Try another suspect.'
        return

    if st.session_state.interrogations >= st.session_state.max_interrogations:
        st.session_state.result_message = 'You have used all your interrogation chances. Make an accusation or start a new case.'
        return

    st.session_state.interrogated.append(index)
    st.session_state.interrogations += 1
    contradiction = find_contradiction(suspect)

    if contradiction and suspect['name'] not in st.session_state.found_contradictions:
        st.session_state.found_contradictions.append(suspect['name'])
        st.session_state.score += 10
        st.session_state.result_message = (
            f'Interrogation result: {suspect["name"]} gave an alibi that contradicts a clue. '\
            f'Contradicting clue: {contradiction["clue"]} '\
            'You earned 10 detective points.'
        )
    elif suspect.get('guilty'):
        st.session_state.result_message = f'{suspect["name"]} looks nervous and avoids eye contact.'
    else:
        st.session_state.result_message = f'{suspect["name"]} appears calm and answers honestly.'


def accuse_index(index):
    if st.session_state.case_over:
        return

    suspect = st.session_state.suspects[index]
    if suspect.get('guilty'):
        st.session_state.result_message = f'Correct! {suspect["name"]} is the murderer. Case closed.'
        st.session_state.case_over = True
    else:
        st.session_state.result_message = f'{suspect["name"]} is not the murderer. Keep investigating or start a new case.'


def card_html(suspect, index):
    alibi_text = suspect['alibi'] if index in st.session_state.interrogated else 'Interrogate to reveal the alibi.'
    seen_text = 'Yes' if index in st.session_state.interrogated else 'No'
    return f'''
<div style="border: 1px solid #666; border-radius: 12px; padding: 16px; margin-bottom: 12px; background: #fbfbfb; box-shadow: 2px 2px 5px rgba(0, 0, 0, 0.08);">
  <h4 style="margin: 0 0 8px 0;">{suspect['name']}</h4>
  <p style="margin: 4px 0;"><strong>Motive:</strong> {suspect['motive']}</p>
  <p style="margin: 4px 0;"><strong>Alibi:</strong> {alibi_text}</p>
  <p style="margin: 4px 0;"><strong>Interrogated:</strong> {seen_text}</p>
</div>
'''


def display_game():
    st.title('🕵️ Detective Mystery')
    st.markdown('### A Streamlit murder mystery case')
    st.markdown(f'**Victim:** {st.session_state.victim}')

    with st.expander('Crime Scene Details', expanded=True):
        st.write(crime_scene['description'])
        st.write(f'Location: {crime_scene["location"]}')

    st.markdown('---')
    st.markdown('### Clues')
    for clue in st.session_state.clues:
        clue_text = clue.get('text') if isinstance(clue, dict) else str(clue)
        st.success(clue_text)

    st.markdown('---')
    st.markdown('### Suspects')
    columns = st.columns(len(st.session_state.suspects))
    for index, suspect in enumerate(st.session_state.suspects):
        with columns[index]:
            st.markdown(card_html(suspect, index), unsafe_allow_html=True)
            if st.button('Interrogate', key=f'interrogate_{index}'):
                interrogate_index(index)
            if st.button('Accuse', key=f'accuse_{index}'):
                accuse_index(index)

    st.markdown('---')
    st.markdown('### Investigation Notes')
    st.info(st.session_state.result_message)


def render_sidebar():
    st.sidebar.title('Detective HQ')
    st.sidebar.markdown('Track your case progress and manage the investigation from the sidebar.')
    st.sidebar.metric('Score', st.session_state.score)
    st.sidebar.metric('Contradictions found', len(st.session_state.found_contradictions))
    st.sidebar.metric('Interrogations left', st.session_state.max_interrogations - st.session_state.interrogations)
    st.sidebar.markdown('---')
    if st.sidebar.button('Start New Case'):
        initialize_game()
        st.experimental_rerun()
    st.sidebar.markdown('**Detective Notes**')
    st.sidebar.write('- Review clues carefully')
    st.sidebar.write('- Interrogate suspects to reveal their alibis')
    st.sidebar.write('- Look for contradictions between alibis and clues')


st.set_page_config(page_title='Detective Mystery', page_icon='🕵️', layout='wide')

if 'game_initialized' not in st.session_state:
    initialize_game()
    st.session_state.game_initialized = True

render_sidebar()
display_game()

